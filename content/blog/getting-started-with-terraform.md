---
title: "Getting Started With Terraform"
date: 2017-08-24T16:23:41+01:00
draft: true
tags: [ "terraform" ]
---

## Definitions

[Terraform](https://www.terraform.io/intro/index.html) is a tool to manage infrastructures.
This means you will create infrastructures (compute instances, storage, networking, DNS entries, ...)
You may be already familiar with [Ansible](https://www.ansible.com/), which is a configuration management tool. 
This means with tools like Ansible, you can configure the infrastructure elements already created either manually or with Terraform.

## What we will do

* Creating an EC2 instance on AWS
* Destroying this EC2 instance

## Install terraform

> Please refer to [the official documentation for variants](https://www.terraform.io/intro/getting-started/install.html). 

```bash
$ brew install terraform
```
## Set up AWS

> Please refer to [the official documentation for variants](http://docs.aws.amazon.com/cli/latest/userguide/installing.html)

If you don't have any AWS account yet, just create one. 
You may be eligible for the free tier. 

```bash
$ pip install awscli --upgrade --user
```
Now you will want to configure it:
```bash
$ aws configure
```
This will prompt your for your account details and store them in your home folder (under `~/.aws/credentials`). 
This part is particularly interesting as most tools around aws pick up these credentials automatically so you don't have to share them in a source control tool.

## Build your first infrastructure

Ok, let's build our first infrastructure! It will just be an EC2 instance, but it's a start.

### Find your image

EC2 instances are built based on an image, which varies per region. You then need to find your one.
The [`describe-images`](http://docs.aws.amazon.com/cli/latest/reference/ec2/describe-images.html) command could be used for that, but there are many many images to choose from and I didn't find any reasonable filtering.

So, the easier way is to:

1. Log in to your console on https://console.aws.amazon.com/ec2/
1. Click "Launch instance'
1. Get the first "Amazon Linux AMI" (you should see a label "Free tier eligible")
1. Copy the image id, here in London region it's `ami-489f8e2c`

Great, now let's get *really* started.

### Create the terraform configuration

Add a new file, call it `demo.ts` with:

```terraform
provider "aws" {
  region = "eu-west-2"
}

resource "aws_instance" "demo" {
  ami           = "ami-489f8e2c"
  instance_type = "t2.micro"
}
```
> Adapt your region and ami id accordingly.

The `provider` block is used to configure the named provider, in our case "aws". 
A provider is responsible for creating and managing resources. 
Multiple provider blocks can exist if a Terraform configuration is composed of multiple providers, which is a common situation.

The `resource` block defines a resource that exists within the infrastructure. 
A resource might be a physical component such as an EC2 instance, or it can be a logical resource such as a Heroku application.

The `resource` block has two strings before opening the block: the `resource type` and the `resource name`. In our example, the resource type is `"aws_instance"` and the name is `"demo"`.
The prefix of the type maps to the provider. 
In our case `"aws_instance"` automatically tells Terraform that it is managed by the `"aws"` provider.

### Initialisation: Terraform init

The `terraform init` command is ran on new projects or after code checkout from repository to install / update plugins and set configuration.

```plain
$ terraform init

Initializing provider plugins...
- Checking for available provider plugins on https://releases.hashicorp.com...
- Downloading plugin for provider "aws" (0.1.4)...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.aws: version = "~> 0.1"

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

### Dry-run: Terraform plan

The `terraform plan` command shows the difference between the current state of your infrastructure and what is described in the files (being your target state).

```plain
$ terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

The Terraform execution plan has been generated and is shown below.
Resources are shown in alphabetical order for quick scanning. Green resources
will be created (or destroyed and then created if an existing resource
exists), yellow resources are being changed in-place, and red resources
will be destroyed. Cyan entries are data sources to be read.

Note: You didn't specify an "-out" parameter to save this plan, so when
"apply" is called, Terraform can't guarantee this is what will execute.

  + aws_instance.demo
      ami:                          "ami-489f8e2c"
      associate_public_ip_address:  "<computed>"
      availability_zone:            "<computed>"
      ebs_block_device.#:           "<computed>"
      ephemeral_block_device.#:     "<computed>"
      instance_state:               "<computed>"
      instance_type:                "t2.micro"
      ipv6_address_count:           "<computed>"
      ipv6_addresses.#:             "<computed>"
      key_name:                     "<computed>"
      network_interface.#:          "<computed>"
      network_interface_id:         "<computed>"
      placement_group:              "<computed>"
      primary_network_interface_id: "<computed>"
      private_dns:                  "<computed>"
      private_ip:                   "<computed>"
      public_dns:                   "<computed>"
      public_ip:                    "<computed>"
      root_block_device.#:          "<computed>"
      security_groups.#:            "<computed>"
      source_dest_check:            "true"
      subnet_id:                    "<computed>"
      tenancy:                      "<computed>"
      volume_tags.%:                "<computed>"
      vpc_security_group_ids.#:     "<computed>"


Plan: 1 to add, 0 to change, 0 to destroy.
```

What does it mean?

* The `+` sign next to `aws_instance.demo` has to be read like a diff, this resource is not currently known by the infrastructure, and Terraform know it has to be created.
This information can also be read at the bottom (`Plan: 1 to add, 0 to change, 0 to destroy.`)
* The `<computed>` values mean that they are not known until the resource has been created.

So at this point, we have one resource to create, which is `aws_instance.demo`. So let's create it!

### Terraform apply

The `terraform apply` command actually applies as its name suggests the infrastructure defined by your code.

```plain
$ terraform apply
aws_instance.demo: Creating...
  ami:                          "" => "ami-489f8e2c"
  associate_public_ip_address:  "" => "<computed>"
  availability_zone:            "" => "<computed>"
  ebs_block_device.#:           "" => "<computed>"
  ephemeral_block_device.#:     "" => "<computed>"
  instance_state:               "" => "<computed>"
  instance_type:                "" => "t2.micro"
  ipv6_address_count:           "" => "<computed>"
  ipv6_addresses.#:             "" => "<computed>"
  key_name:                     "" => "<computed>"
  network_interface.#:          "" => "<computed>"
  network_interface_id:         "" => "<computed>"
  placement_group:              "" => "<computed>"
  primary_network_interface_id: "" => "<computed>"
  private_dns:                  "" => "<computed>"
  private_ip:                   "" => "<computed>"
  public_dns:                   "" => "<computed>"
  public_ip:                    "" => "<computed>"
  root_block_device.#:          "" => "<computed>"
  security_groups.#:            "" => "<computed>"
  source_dest_check:            "" => "true"
  subnet_id:                    "" => "<computed>"
  tenancy:                      "" => "<computed>"
  volume_tags.%:                "" => "<computed>"
  vpc_security_group_ids.#:     "" => "<computed>"
aws_instance.demo: Still creating... (10s elapsed)
aws_instance.demo: Still creating... (20s elapsed)
aws_instance.demo: Creation complete (ID: i-0cfedba420a5fd163)

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```
Here we are! Let's do a `terraform plan` again to see the difference:

```plain
$ terraform plan
terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

aws_instance.demo: Refreshing state... (ID: i-0cfedba420a5fd163)
No changes. Infrastructure is up-to-date.

This means that Terraform did not detect any differences between your
configuration and real physical resources that exist. As a result, Terraform
doesn't need to do anything.
```
**No changes. Infrastructure is up-to-date.** Well, that's what we expected.

You can go ahead and check your instance in the AWS console, you'll see it running.

Alternatively, you can use `aws-cli` with:

```bash
$ aws ec2 describe-instances --filters "Name=instance-state-name,Values=running,Name=image-id,Values=ami-489f8e2c"
```
Or even, as the terraform plan gives you the instance id, you could:
```bash
$ aws ec2 get-console-output --instance-id i-0cfedba420a5fd163
```
### Terraform state

Look at your files directory now, you should have a new one called `terraform.tfstate`.
This file keeps track of your infrastructure state, it's extremely important! That's even one of the key features of Terraform, to keep track of your infrastructure's state.
> Note that if you're working in a team, this file will likely be frequently under merge conflicts.
Tt's best practice to use a ["remote state"](https://www.terraform.io/docs/state/remote.html) instead, which is a sort of backend for your terraform, storing state on a deported S3 for example.

### Terraform show

Let's see what terraform can tell us about our infrastructure now:

```plain
$ terraform show
terraform show
aws_instance.demo:
  id = i-0cfedba420a5fd163
  ami = ami-489f8e2c
  associate_public_ip_address = true
  availability_zone = eu-west-2a
  disable_api_termination = false
  ebs_block_device.# = 0
  ebs_optimized = false
  ephemeral_block_device.# = 0
  iam_instance_profile = 
  instance_state = running
  instance_type = t2.micro
  ipv6_addresses.# = 0
  key_name = 
  monitoring = false
  network_interface.# = 0
  network_interface_id = eni-9891dde2
  primary_network_interface_id = eni-9891dde2
  private_dns = ip-172-31-14-93.eu-west-2.compute.internal
  private_ip = 172.31.14.93
  public_dns = ec2-35-176-145-154.eu-west-2.compute.amazonaws.com
  public_ip = 35.176.145.154
  root_block_device.# = 1
  root_block_device.0.delete_on_termination = true
  root_block_device.0.iops = 100
  root_block_device.0.volume_size = 8
  root_block_device.0.volume_type = gp2
  security_groups.# = 0
  source_dest_check = true
  subnet_id = subnet-99cb7be2
  tags.% = 0
  tenancy = default
  volume_tags.% = 0
  vpc_security_group_ids.# = 1
  vpc_security_group_ids.3104331321 = sg-238acd4a
```

### Clean up: Terraform destroy

Say this infrastructure was used to run tests and are part of a Jenkins pipeline for instance. 
Now that the tests are complete, we can dispose of it.

The `terraform plan` command can again be used as a dry-run to see if it will do want we really want it to do.

```plain
terraform plan -destroy
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

aws_instance.demo: Refreshing state... (ID: i-0cfedba420a5fd163)
The Terraform execution plan has been generated and is shown below.
Resources are shown in alphabetical order for quick scanning. Green resources
will be created (or destroyed and then created if an existing resource
exists), yellow resources are being changed in-place, and red resources
will be destroyed. Cyan entries are data sources to be read.

Note: You didn't specify an "-out" parameter to save this plan, so when
"apply" is called, Terraform can't guarantee this is what will execute.

  - aws_instance.demo


Plan: 0 to add, 0 to change, 1 to destroy.
```
Apparently here, 1 resource would be destroyed, which is our `aws_instance.demo` and turns out it's actually what we need.
Great, so let's do it!

```plain
terraform destroy
aws_instance.demo: Refreshing state... (ID: i-0cfedba420a5fd163)

The Terraform destroy plan has been generated and is shown below.
Resources are shown in alphabetical order for quick scanning.
Resources shown in red will be destroyed.

  - aws_instance.demo


Do you really want to destroy?
  Terraform will delete all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

aws_instance.demo: Destroying... (ID: i-0cfedba420a5fd163)
aws_instance.demo: Still destroying... (ID: i-0cfedba420a5fd163, 10s elapsed)
aws_instance.demo: Still destroying... (ID: i-0cfedba420a5fd163, 20s elapsed)
aws_instance.demo: Still destroying... (ID: i-0cfedba420a5fd163, 30s elapsed)
aws_instance.demo: Still destroying... (ID: i-0cfedba420a5fd163, 40s elapsed)
aws_instance.demo: Still destroying... (ID: i-0cfedba420a5fd163, 50s elapsed)
aws_instance.demo: Destruction complete

Destroy complete! Resources: 1 destroyed.
```
> Note that there's a prompt asking you to confirm you want to destroy your infrastructure.
The `terraform destroy` accepts a `-force` argument to skip the confirmation prompt.  

## Source code

You can find the code used in this article at: https://github.com/iyp-uk/terraform-getting-started
