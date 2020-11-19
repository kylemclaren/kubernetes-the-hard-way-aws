# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster.

> Ensure a default compute region has been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud (VPC)

In this section a dedicated [Virtual Private Cloud](http://aws.amazon.com/documentation/vpc) (VPC) network will be setup to host the Kubernetes cluster.

Create the custom VPC network and subnet. A [subnet](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-vpcs.html#AddaSubnets) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Here we create the VPC and specify a custom CIDR block for the subnet, in one step:

```sh
aws ec2 create-vpc \
  --cidr-block 10.240.0.0/24
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

> output

```json
{
    "Vpc": {
        "CidrBlock": "10.240.0.0/24",
        "DhcpOptionsId": "dopt-cad0a1b0",
        "State": "pending",
        "VpcId": "vpc-0f4191f90bd8c4e71",
        "OwnerId": "<AWS_ACCOUNT_ID>",
        "InstanceTenancy": "default",
        "Ipv6CidrBlockAssociationSet": [],
        "CidrBlockAssociationSet": [
            {
                "AssociationId": "vpc-cidr-assoc-0c26451137d95ec4c",
                "CidrBlock": "10.240.0.0/24",
                "CidrBlockState": {
                    "State": "associated"
                }
            }
        ],
        "IsDefault": false
    }
```

You'll want to note down the `VpcId` for later use.

Next, we create a subnet for the VPC we just, using the same CIDR block:

```sh
aws ec2 create-subnet \
 --vpc-id vpc-0f4191f90bd8c4e71 \
 --cidr-block 10.240.0.0/24
```

```sh
aws ec2 describe-subnets --filters Name="subnet-id",Values="subnet-0e41b90871027db5b"
```

### Firewall Rules

When dealing with Firewalls in AWS, you are usually working with the concept of a "Security Group".

Create a new security group and associate it with the VPC we created in the previous step

```sh
aws ec2 create-security-group --group-name kubernetes-the-hard-way --description "Security group for Kubernetes the Hard Way cluster" --vpc-id vpc-0f4191f90bd8c4e71
```

Note the `GroupId` in the output as we'll need it to create the firewall rules.

Create an ingress firewall rule that allows external SSH, ICMP, and HTTPS on all IP address ranges:

```sh
aws ec2 authorize-security-group-ingress --group-id sg-0bd79e2e8238927ec --ip-permissions IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges='[{CidrIp=0.0.0.0/0}]' IpProtocol=tcp,FromPort=6443,ToPort=6443,IpRanges='[{CidrIp=0.0.0.0/0}]' IpProtocol=icmp,FromPort=-1,ToPort=-1,IpRanges='[{CidrIp=0.0.0.0/0}]'
```

> An [external load balancer](https://cloud.google.com/compute/docs/load-balancing/network/) will be used to expose the Kubernetes API Servers to remote clients.

> An [external network load balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html/) will be used to expose the Kubernetes API Servers to remote clients.

List the firewall rules in our VPC network security group:

```sh
aws ec2 describe-security-groups --group-ids sg-0bd79e2e8238927ec
```

> output

```json
{
  "SecurityGroups": [
    {
      "Description": "Security group for Kubernetes the Hard Way cluster",
      "GroupName": "kubernetes-the-hard-way",
      "IpPermissions": [
        {
          "FromPort": 6443,
          "IpProtocol": "tcp",
          "IpRanges": [
            {
              "CidrIp": "0.0.0.0/0"
            }
          ],
          "Ipv6Ranges": [],
          "PrefixListIds": [],
          "ToPort": 6443,
          "UserIdGroupPairs": []
        },
        {
          "FromPort": 22,
          "IpProtocol": "tcp",
          "IpRanges": [
            {
              "CidrIp": "0.0.0.0/0"
            }
          ],
          "Ipv6Ranges": [],
          "PrefixListIds": [],
          "ToPort": 22,
          "UserIdGroupPairs": []
        },
        {
          "FromPort": -1,
          "IpProtocol": "icmp",
          "IpRanges": [
            {
              "CidrIp": "0.0.0.0/0"
            }
          ],
          "Ipv6Ranges": [],
          "PrefixListIds": [],
          "ToPort": -1,
          "UserIdGroupPairs": []
        }
      ],
      "OwnerId": "365014073076",
      "GroupId": "sg-0bd79e2e8238927ec",
      "IpPermissionsEgress": [
        {
          "IpProtocol": "-1",
          "IpRanges": [
            {
              "CidrIp": "0.0.0.0/0"
            }
          ],
          "Ipv6Ranges": [],
          "PrefixListIds": [],
          "UserIdGroupPairs": []
        }
      ],
      "VpcId": "vpc-0f4191f90bd8c4e71"
    }
  ]
}
```

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers. Within AWS, this is know as an Elastic IP:

```sh
aws ec2 allocate-address \
    --domain vpc
```

> note the `AllocationId`


> output

```
NAME                     ADDRESS/RANGE   TYPE      PURPOSE  NETWORK  REGION    SUBNET  STATUS
kubernetes-the-hard-way  XX.XXX.XXX.XXX  EXTERNAL                    us-west1          RESERVED
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 20.04 (`ami-0885b1f6bd170450c`), which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

First, create a key pair to provide us with ssh access to the control plan nodes. We'll save the file to our local directory:

```sh
aws ec2 create-key-pair --key-name controlPlane --output text > controlPlane.pem
```

Create three compute instances which will host the Kubernetes control plane:

```sh
for i in 0 1 2; do
  aws ec2 run-instances \
      --image-id ami-0885b1f6bd170450c \
      --instance-type t2.micro \
      --count 1 \
      --subnet-id subnet-0e41b90871027db5b \
      --key-name k8sHardWay \
      --security-group-ids sg-0bd79e2e8238927ec \
      --private-ip-address 10.240.0.1${i} \
      --block-device-mappings 'DeviceName=/dev/sdh,Ebs={DeleteOnTermination=true,VolumeSize=100}' \
      --tag-specifications "ResourceType=instance,Tags=[{Key=project,Value=kubernetes-the-hard-way},{Key=nodeType,Value=controlPlane},{Key=Name,Value=controller${i}}]" "ResourceType=volume,Tags=[{Key=project,Value=kubernetes-the-hard-way},{Key=nodeType,Value=controlPlane}]"
done
```
aws ec2 run-instances \
    --image-id ami-0885b1f6bd170450c \
    --instance-type t2.micro \
    --count 1 \
    --subnet-id subnet-0e41b90871027db5b \
    --key-name controlPlane \
    --security-group-ids sg-0bd79e2e8238927ec \
    --private-ip-address 10.240.0.1${i} \
    --block-device-mappings 'DeviceName=/dev/sdh,Ebs={DeleteOnTermination=true,VolumeSize=100}' \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=project,Value=kubernetes-the-hard-way},{Key=nodeType, Value=controlPlane}]' 'ResourceType=volume,Tags=[{Key=project,Value=kubernetes-the-hard-way},{Key=nodeType, Value=controlPlane}]'
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-standard-2 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

### Verification

List the compute instances in your default compute zone:

```
gcloud compute instances list --filter="tags.items=kubernetes-the-hard-way"
```

> output

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
controller-0  us-west1-c  e2-standard-2               10.240.0.10  XX.XX.XX.XXX   RUNNING
controller-1  us-west1-c  e2-standard-2               10.240.0.11  XX.XXX.XXX.XX  RUNNING
controller-2  us-west1-c  e2-standard-2               10.240.0.12  XX.XXX.XX.XXX  RUNNING
worker-0      us-west1-c  e2-standard-2               10.240.0.20  XX.XX.XXX.XXX  RUNNING
worker-1      us-west1-c  e2-standard-2               10.240.0.21  XX.XX.XX.XXX   RUNNING
worker-2      us-west1-c  e2-standard-2               10.240.0.22  XX.XXX.XX.XX   RUNNING
```

## Configuring SSH Access

SSH will be used to configure the controller and worker instances. When connecting to compute instances for the first time SSH keys will be generated for you and stored in the project or instance metadata as described in the [connecting to instances](https://cloud.google.com/compute/docs/instances/connecting-to-instance) documentation.

Test SSH access to the `controller-0` compute instances:

```
gcloud compute ssh controller-0
```

If this is your first time connecting to a compute instance SSH keys will be generated for you. Enter a passphrase at the prompt to continue:

```
WARNING: The public SSH key file for gcloud does not exist.
WARNING: The private SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: SSH keygen will be executed to generate a key.
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

At this point the generated SSH keys will be uploaded and stored in your project:

```
Your identification has been saved in /home/$USER/.ssh/google_compute_engine.
Your public key has been saved in /home/$USER/.ssh/google_compute_engine.pub.
The key fingerprint is:
SHA256:nz1i8jHmgQuGt+WscqP5SeIaSy5wyIJeL71MuV+QruE $USER@$HOSTNAME
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|                 |
|                 |
|        .        |
|o.     oS        |
|=... .o .o o     |
|+.+ =+=.+.X o    |
|.+ ==O*B.B = .   |
| .+.=EB++ o      |
+----[SHA256]-----+
Updating project ssh metadata...-Updated [https://www.googleapis.com/compute/v1/projects/$PROJECT_ID].
Updating project ssh metadata...done.
Waiting for SSH key to propagate.
```

After the SSH keys have been updated you'll be logged into the `controller-0` instance:

```
Welcome to Ubuntu 20.04 LTS (GNU/Linux 5.4.0-1019-gcp x86_64)
...
```

Type `exit` at the prompt to exit the `controller-0` compute instance:

```
$USER@controller-0:~$ exit
```

> output

```
logout
Connection to XX.XX.XX.XXX closed
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
