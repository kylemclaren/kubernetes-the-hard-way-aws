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
aws ec2 describe-subnets \
  --filters Name="subnet-id",Values="subnet-0e41b90871027db5b"
```

### Firewall Rules

When dealing with Firewalls in AWS, you are usually working with the concept of a "Security Group".

Create a new security group and associate it with the VPC we created in the previous step

```sh
aws ec2 create-security-group \
  --group-name kubernetes-the-hard-way \
  --description "Security group for Kubernetes the Hard Way cluster" \
  --vpc-id vpc-0f4191f90bd8c4e71
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
aws ec2 describe-security-groups \
  --group-ids sg-0bd79e2e8238927ec
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
aws ec2 create-key-pair \
  --key-name k8sHardWay \
  --output text > k8sHardWay.pem
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
    --block-device-mappings "DeviceName=/dev/sdh,Ebs={DeleteOnTermination=true,VolumeSize=100}" \
    --tag-specifications "ResourceType=instance,Tags=[{Key=project,Value=kubernetes-the-hard-way},{Key=nodeType,Value=controlPlane},{Key=Name,Value=controller-${i}}]" "ResourceType=volume,Tags=[{Key=project,Value=kubernetes-the-hard-way},{Key=nodeType,Value=controlPlane}]" \
    > /dev/null
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```sh
for i in 0 1 2; do
  aws ec2 run-instances \
    --image-id ami-0885b1f6bd170450c \
    --instance-type t2.micro \
    --count 1 \
    --subnet-id subnet-0e41b90871027db5b \
    --key-name k8sHardWay \
    --security-group-ids sg-0bd79e2e8238927ec \
    --private-ip-address 10.240.0.2${i} \
    --user-data pod-cidr=10.200.${i}.0/24 \
    --block-device-mappings "DeviceName=/dev/sdh,Ebs={DeleteOnTermination=true,VolumeSize=100}" \
    --tag-specifications "ResourceType=instance,Tags=[{Key=project,Value=kubernetes-the-hard-way},{Key=nodeType, Value=worker},{Key=Name,Value=worker-${i}}]" "ResourceType=volume,Tags=[{Key=project,Value=kubernetes-the-hard-way},{Key=nodeType, Value=worker}]" \
    > /dev/null
done
```

### Verification

List the compute instances we've just created:

```sh
aws ec2 describe-instances \
  --filters Name=tag:project,Values=kubernetes-the-hard-way \
  --query 'Reservations[*].Instances[*].{Instance:InstanceId,AZ:Placement.AvailabilityZone,Name:Tags[?Key==`Name`]|[0].Value,InternalIP:PrivateIpAddress,ExternalIP:PublicIpAddress,Status:State.Name}' \
  --output table
```

> output

```txt
-----------------------------------------------------------------------------------------------
|                                      DescribeInstances                                      |
+------------+-------------+----------------------+--------------+----------------+-----------+
|     AZ     | ExternalIP  |      Instance        | InternalIP   |     Name       |  Status   |
+------------+-------------+----------------------+--------------+----------------+-----------+
|  us-east-1f|  None       |  i-003c3bd0a883e9b3b |  10.240.0.11 |  controller-1  |  running  |
|  us-east-1f|  None       |  i-0459bfbc79de2877b |  10.240.0.12 |  controller-2  |  running  |
|  us-east-1f|  None       |  i-0b9d0a54b268d3d42 |  10.240.0.10 |  controller-0  |  running  |
|  us-east-1f|  None       |  i-045ed5daf55a9a47b |  10.240.0.20 |  worker-0      |  running  |
|  us-east-1f|  None       |  i-015ac68455ba3b1d5 |  10.240.0.21 |  worker-1      |  running  |
|  us-east-1f|  None       |  i-09f7aa65a9df1b4a9 |  10.240.0.22 |  worker-2      |  running  |
+------------+-------------+----------------------+--------------+----------------+-----------+
```

## SSH Access

SSH will be used to configure the controller and worker instances. Use the `.pem` key we created with the `create-key-pair` command earlier to auth t the instances. First though, some housekeeping with permisisons for the key file:

```sh
chmod 400 k8sHardWay.pem
```

Test SSH access to the `controller-0` compute instances:

```sh
ssh -i "k8sHardWay.pem" ubuntu@ec2-your-external-ip-address.us-east-1.compute.amazonaws.com
```

If this is your first time connecting to a compute instance, you'll be asked to X*X_X*. You'll then be logged into the `controller-0` instance:

```sh
Welcome to Ubuntu 20.04 LTS (GNU/Linux 5.4.0-1019-gcp x86_64)
...
```

Type `exit` at the prompt to exit the `controller-0` compute instance:

```sh
$USER@controller-0:~$ exit
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
