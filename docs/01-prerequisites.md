# Prerequisites

## Amazon Web Services

This tutorial leverages the Amazon Web Services (AWS) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. [Sign up](https://aws.amazon.com/) and enjoy 12 months of Free Tier usage.

<!-- [Estimated cost](https://cloud.google.com/products/calculator#id=873932bc-0840-4176-b0fa-a8cfd4ca61ae) to run this tutorial:  -->

> The compute resources required for this tutorial may exceed the AWS free tier.

## AWS CLI

### Install the AWS CLI version 2

Follow the AWS CLI [documentation](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html/) to install and configure the `aws` command line utility.

If you're on macOS and use Homebrew, simply run:

```sh
brew install awscli
```

Verify the AWS CLI version is 2.1.1 or higher:

```sh
aws --version
```

### Set Access Credentials and Default Region

If you are using the `aws` command-line tool for the first time `configure` is the easiest way to do this:

```sh
aws configure
```

You will need to get your credentials for AWS which incude an **Access Key ID** and a **Secret Access Key**. It is highly recommended to use IAM and not root credentials, however, that is beyond the scope of this document for now.

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with synchronize-panes enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable synchronize-panes by pressing `ctrl+b` followed by `shift+:`. Next type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)
