>> terraform

1) Creating the project directory
In a location of your choice, create a directory named 1-ec2-instance

2) Create the following directory structure (where the .tf files are blank text files):

1-ec2-instance/
    - providers.tf
    - main.tf
    - aws_ami.tf
    - variables.tf
    - .gitignore
	
Note: This is not a required directory structure. Terraform will automatically read all `.tf` files within the directory and figure out what to do. This is a file structure that has proven effective in a production environment.


3) .gitignore
I would recommend creating a Git repository with these files. If you do so, you should start with this .gitignore content:

# Compiled files
*.tfstate
*.tfstate.backup

# Module directory
.terraform/

# Sensitive Files
/variables.tf

I recommend adding /variables.tf to your .gitignore file because we're about to put some AWS secrets into it. Keeping the secrets out of Github can keep them more secure and resistent to accidents. In the future, your team may also want to use different secrets to manage permissions to different resources.

If working with a team, you can choose how you'd like these variables to be shared between each member of the team in a way that's right for you.


4) variables.tf
Here is where we'll set some variables to be re-used by the rest of the configuration. It will also serve as a handy place to keep our AWS secrets.

Let's start with these contents for the variables.tf file:

# AWS Config

variable "aws_access_key" {
  default = "YOUR_ADMIN_ACCESS_KEY"
}

variable "aws_secret_key" {
  default = "YOUR_ADMIN_SECRET_KEY"
}

variable "aws_region" {
  default = "us-west-2"
}


5) providers.tf
This file is pretty short:

provider "aws" {
  access_key = "${var.aws_access_key}"
  secret_key = "${var.aws_secret_key}"
  region     = "${var.aws_region}"
  
  version = "~> 1.7"
}


In Terraform, Providers are interfaces to the services that maintain our Resources. For example - An EC2 Instance is a Resource provided by the Amazon Web Services Provider. A Git Repository is a Resource provided by the Github Provider.

Because Terraform is an open source tool, contributors can build custom providers to accomplish different tasks. For now, we will focus purely on the AWS provider and the resources it provides.


6) aws_ami.tf
This file is dedicated to finding the right Ubuntu AMI to install on our server. AMI IDs change from region to region and change over time as upgrades come out. We're going to create a data source to track down the right one.

The contents of the aws_ami.tf file are:

data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-trusty-14.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"] # Canonical
}



7) main.tf
Here's the fun part. The part that initializes the server. It's also surprisingly short:

resource "aws_instance" "my-test-instance" {
  ami             = "${data.aws_ami.ubuntu.id}"
  instance_type   = "t2.micro"

  tags {
    Name = "test-instance"
  }
}



With all the work we've done in the other files, all we need to do here is describe the server we want.

Let's break down what this configuration is saying:

We are defining an aws_instance with the unique Terraform identifier of my-test-instance

That instance should use the AMI found in aws_ami.tf to initialize the server

That instance should be a t2.micro (the cheapest AWS instance type)

We've attached a Name tag to the instance, test-instance, for easy identification



8) Creating the Infrastructure
Open up bash, navigate to the project's directory, and run the following:

$ terraform init
This will download and install the proper version of the AWS provider for your project and place it in a directory called .terraform.



We'll now run the command that will take the configurations we've written and use the AWS API to build our servers. This command is one that you'll be using throughout most of your time with Terraform:

$ terraform apply



Enter a value:
A message like this will always appear before changes are made to your infrastructure through apply. Terraform analyzes the existing resources in your AWS account and builds a plan of exactly what it will do and why. It outputs this plan and asks whether or not you'd like to make the changes.

Note:

Always read this plan carefully and take note of what's being created, modified or destroyed. This will prevent you from accidentally destroying infrastructure that does not need to be modified.

You can type yes and hit Enter to create the new server.

Congratulations! You've created your first piece of AWS infrastructure through Terraform. Welcome to the wonderful world of Infrastructure as Code!



>> Destroying the Infrastructure
The idea of destroying infrastructure can sound a bit ominous. But, we're going to start getting rid of that ominous feeling right now. Let's destroy this server we've created!

With an established infrastructure, you are unlikely to use this next command. But destruction of resources will happen on a smaller scale, implicitly, through certain configuration changes.

Until we make this server do something in future chapters, let's destroy it to save a little money. Run the following command:

$ terraform destroy



>> Attaching a Static IP
In order to set up a proper domain name, a static IP address is often required. To make the rest of the steps easier - Let's start off by attaching a static IP address to our EC2 instance.

Let's create a file called eip.tf to store our Elastic IP resource configuration in. Filling it with the following contents:

resource "aws_eip" "test-eip" {
  instance    = "${aws_instance.my-test-instance.id}"
}




>> Creating the Security Groups
Let's create a file named security_group.tf with the contents:

resource "aws_security_group" "allow_ssh" {
  name        = "allow-ssh"
  description = "Allow SSH inbound traffic"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "allow_outbound" {
  name        = "allow-all-outbound"
  description = "Allow all outbound traffic"

  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

Here we're creating an aws_security_group resource with the identifier allow_ssh. This security group allows incoming (ingress) traffic to port 22 (the SSH port) from the CIDR Subnet Mask 0.0.0.0/0 (all IPs).

We are also creating a security group for outbound (egress) connections to any host any port, through allow_outbound. We're doing that so that our server can download RVM and Rails from the proper servers.



>> Attaching the Security Groups
Let's modify the main.tf file to contain:

resource "aws_instance" "my-test-instance" {
  ami             = "${data.aws_ami.ubuntu.id}"
  instance_type   = "t2.micro"

  security_groups = [
    "${aws_security_group.allow_ssh.name}",
    "${aws_security_group.allow_outbound.name}"
  ]

  tags {
    Name = "test-instance"
  }
}

This will attach the security groups we just created, to our EC2 instance. Allowing it to receive incoming connections on port 22 and allowing all outbound connections from the server.
