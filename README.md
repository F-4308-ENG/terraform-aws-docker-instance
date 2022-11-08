# Hands-on Terraform-05: Publishing Modules and Terraform Workspaces

Purpose of the this hands-on training is to give students the knowledge of publishing modules and using workspaces in Terraform.

## Learning Outcomes

At the end of the this hands-on training, students will be able to;

- Publish Terraform Modules

- Use Terraform Workspaces

## Outline

- Part 1 - Publish Terraform Modules

- Part 2 - Use Terraform Workspaces

## Part 1 - Publish Terraform Modules

- Anyone can publish and share modules on the Terraform Registry.

- Published modules support versioning, automatically generate documentation, allow browsing version histories, show examples and READMEs, and more. Terraform recommend publishing reusable modules to a registry.

- Public modules are managed via ``Git`` and ``GitHub``. Once a module is published, you can release a new version of a module by simply pushing a properly formed Git tag.

### Requirements

- The list below contains all the requirements for publishing a module:

* ``GitHub``. The module must be on GitHub and must be a ``public`` repo. This is only a requirement for the public registry. If you're using a private registry, you may ignore this requirement.

* ``Named`` terraform-<PROVIDER>-<NAME>. Module repositories must use this three-part name format, where <NAME> reflects the type of infrastructure the module manages and <PROVIDER> is the main provider where it creates that infrastructure. The <NAME> segment can contain additional hyphens. Examples: terraform-google-vault or terraform-aws-ec2-instance.

* ``Repository description``. The GitHub repository description is used to populate the short description of the module. This should be a simple one sentence description of the module.

* ``Standard module structure``. The module must adhere to the standard module structure. This allows the registry to inspect your module and generate documentation, track resource usage, parse submodules and examples, and more.

* ``x.y.z tags for releases``. The registry uses tags to identify module versions. Release tag names must be a semantic version, which can optionally be prefixed with a v. For example, v1.0.4 and 0.9.2. To publish a module initially, at least one release tag must be present. Tags that don't look like version numbers are ignored. (https://semver.org/)

- source link: https://www.terraform.io/registry/modules/publish

### Create a module to create an aws instance with amazon linux 2 ami (kernel 5.10).

- Create a directory for modules to publish.

```bash
cd && mkdir clarusway-modules && cd clarusway-modules && touch main.tf variables.tf outputs.tf versions.tf userdata.sh README.md .gitignore
```

- Go to the `versions.tf` and copy the latest provider version from the terraform documentaion (https://registry.terraform.io/providers/hashicorp/aws/latest/docs).

```go
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}
```

- Go to the `variables.tf` and prepare your modules variables.

```go
variable "instance_type" {
  type = string
  default = "t2.micro"
}

variable "key_name" {
  type = string
}

variable "num_of_instance" {
  type = number
  default = 1
}

variable "tag" {
  type = string
  default = "Docker-Instance"
}

variable "server-name" {
  type = string
  default = "docker-instance"
}

variable "docker-instance-ports" {
  type = list(number)
  description = "docker-instance-sec-gr-inbound-rules"
  default = [22, 80, 8080]
}
```

- Go to the `main.tf` and prepare a config file to create an aws intance with amazon linux 2 ami (kernel 5.10).

```go
data "aws_ami" "amazon-linux-2" {
  owners      = ["amazon"]
  most_recent = true

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }

  filter {
    name   = "owner-alias"
    values = ["amazon"]
  }

  filter {
    name   = "name"
    values = ["amzn2-ami-kernel-5.10-hvm*"]
  }
}

data "template_file" "userdata" {
  template = file("${abspath(path.module)}/userdata.sh")
  vars = {
    server-name = var.server-name
  }
}

resource "aws_instance" "tfmyec2" {
  ami = data.aws_ami.amazon-linux-2.id
  instance_type = var.instance_type
  count = var.num_of_instance
  key_name = var.key_name
  vpc_security_group_ids = [aws_security_group.tf-sec-gr.id]
  user_data = data.template_file.userdata.rendered
  tags = {
    Name = var.tag
  }
}

resource "aws_security_group" "tf-sec-gr" {
  name = "${var.tag}-terraform-sec-grp"
  tags = {
    Name = var.tag
  }

  dynamic "ingress" {
    for_each = var.docker-instance-ports
    iterator = port
    content {
      from_port = port.value
      to_port = port.value
      protocol = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port =0
    protocol = "-1"
    to_port =0
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

- Go to the `outputs.tf` and write some outputs.

```go
output "instance_public_ip" {
  value = aws_instance.tfmyec2.*.public_ip
}

output "sec_gr_id" {
  value = aws_security_group.tf-sec-gr.id
}

output "instance_id" {
  value = aws_instance.tfmyec2.*.id
}
```

- Go to the `userdata.sh` file and write the followings.

```bash
#!/bin/bash
hostnamectl set-hostname ${server-name}
yum update -y
amazon-linux-extras install docker -y
systemctl start docker
systemctl enable docker
usermod -a -G docker ec2-user
# install docker-compose
curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" \
-o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

- Go to the `.gitignore` file and write the followings. 

```bash
# Local .terraform directories
**/.terraform/*

# Terraform lockfile
.terraform.lock.hcl

# .tfstate files
*.tfstate
*.tfstate.*

# Crash log files
crash.log

# Exclude all .tfvars files, which are likely to contain sentitive data, such as
# password, private keys, and other secrets. These should not be part of version
# control as they are data points which are potentially sensitive and subject
# to change depending on the environment.
*.tfvars

```

- Go to the `README.md` and make a description of your module.

---
Terraform Module to provision an AWS EC2 instance with the latest amazon linux 2 ami and installed docker in it.

Not intended for production use. It is an example module.

It is just for showing how to create a publish module in Terraform Registry.

Usage:

```hcl

provider "aws" {
  region = "us-east-1"
}

module "docker_instance" {
    source = "<github-username>/docker-instance/aws"
    key_name = "clarusway"
}