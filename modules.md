# What are Modules?
We've created a number of resources over the past few chapters. EC2 Instances, Elastic IP addresses, Security Groups, SSH Key Pairs... We have all those things in the root directory.

If we want to make large changes to any of these, we have to jump between a lot of files. If we want to create another server with the same configuration, we need to duplicate all of that code.

As an infrastructure becomes larger, you can imagine this would get pretty hard to manage!

This is where modules come to the rescue! Modules essentially allow us to take a collection of resources and data sources and create a larger, custom resource out of the combination. We can define input variables and output variables to this module so that we can change large parts of the infrastructure with very little effort and then pull out all of the useful information.

Modules are basically your dreams, come to life.

# Initial setup
Let's start off by copying the project directory from the last chapter:

$ cp -R 2-installing-rails/ 3-5-modules/
We'll then make a directory called rails-server and move some files into it:

$ mkdir rails-server
$ mv aws_ami.tf rails-server/
$ mv eip.tf rails-server/
$ mv main.tf rails-server/
$ mv outputs.tf rails-server/
$ rm key_pair.tf

# Input Variables
Modules can take input variables that modify the behavior of the resources they contain. We describe these variables through a variables.tf file and define their values as attributes when we instantiate the module.

Let's create a file in the location of rails-server/variables.tf:

variable "instance_type" {
  type = "string"
  default = "t2.micro"
}

variable "name" {
  type = "string"
}

variable "key_pair" {
  type = "string"
  default = "my_test_key"
}

variable "key_pair_key" {
  type = "string"
}

variable "security_groups" {
  type = "list"
  default = []
}
There's no need to .gitignore this file, like we did the other variables.tf. It doesn't contain any secret keys or passwords.
Now that we have these variables, we can make use of them by modifying the file rails-server/main.tf:

resource "aws_instance" "my-test-instance" {
  ami             = "${data.aws_ami.ubuntu.id}"
  instance_type   = "${var.instance_type}"
  key_name        = "${var.key_pair}"

  security_groups = ["${var.security_groups}"]

  provisioner "remote-exec" {
    inline = [
      "command curl -sSL https://rvm.io/mpapis.asc | gpg --import -",
      "\\curl -sSL https://get.rvm.io | bash -s stable --rails",
    ]

    connection {
      type          = "ssh"
      user          = "ubuntu"
      private_key   = "${file("${var.key_pair_key}")}"
    }
  }

  tags {
    Name = "${var.name}"
  }
}

These variables are used just like the variables we used to define our AWS secrets and give them to the AWS provider.

# A few things to note:

A variable with the type list can be used by wrapping it in square brackets (["${var.some_var}"])

These variables are not global and are in a different scope from the variables used in the parent infrastructure.

Defaults for variables can be set to make interacting with the module less verbose

# Instantiating Modules
It's time to create a new main.tf file in root directory of 3-5-modules/. This time we'll make use of the module we've just built instead of the raw AWS resources.

We'll actually make two servers to prepare us for running a production application:

resource "aws_key_pair" "my-rails-key" {
  key_name   = "my_rails_key"
  public_key = "${file("my_test_key.pub")}"
}

module "my-rails-server" {
  source = "rails-server"

  name            = "my-rails-server"
  key_pair        = "${aws_key_pair.my-rails-key.key_name}"
  key_pair_key    = "~/.ssh/my_test_key"
  security_groups = [
    "${aws_security_group.allow_ssh.name}",
    "${aws_security_group.allow_outbound.name}"
  ]
}

module "my-rails-server-2" {
  source = "rails-server"

  name            = "my-rails-server-2"
  key_pair        = "${aws_key_pair.my-rails-key.key_name}"
  key_pair_key    = "~/.ssh/my_test_key"
  security_groups = [
    "${aws_security_group.allow_ssh.name}",
    "${aws_security_group.allow_outbound.name}"
  ]
}

# Simply run terraform init and terraform apply to create your new servers!

# What we've done
There are a few things happening here:

We start off by making a key pair for connecting to the new servers. This is placed outside of the modules so that only one aws_key_pair resource is created (instead of two).

We create my-rails-server and my-rails-server-2 using the new rails-server module we've created. The module source attribute is simply the path to the directory that stores the module files.

We've used the variables we defined in rails-server/variables.tf to easily configure the nested resources within each instantiated module.

# Output Variables
Let's create a root outputs.tf file to create some useful CLI outputs for our new servers.

# Notice that we moved the old outputs.tf file to rails-server/outputs.tf in the initial setup. That file had defined an output named server-ip that we can now utilize on our instantiated modules.

To see the value of these variable in the CLI, we can create an outputs.tf in the root 3-5-modules/ folder so that Terraform will output the module attributes we care about:

output "server-ip-1" {
  value = "${module.my-rails-server.server-ip}"
}

output "server-ip-2" {
  value = "${module.my-rails-server-2.server-ip}"
}
view raw3-5-abstraction-outputs.tf hosted with ❤ by GitHub
Running terraform apply one last time will give us what we're interested in!

aws_key_pair.my-rails-key: Refreshing state... (ID: my_rails_key)
aws_security_group.allow_ssh: Refreshing state... (ID: sg-f067688c)
aws_security_group.allow_outbound: Refreshing state... (ID: sg-62646b1e)
data.aws_ami.ubuntu: Refreshing state...
data.aws_ami.ubuntu: Refreshing state...
aws_instance.my-test-instance: Refreshing state... (ID: i-0ebf9d64f80fe9912)
aws_instance.my-test-instance: Refreshing state... (ID: i-0da963040524110f7)
aws_eip.test-eip: Refreshing state... (ID: eipalloc-c9edf9f4)
aws_eip.test-eip: Refreshing state... (ID: eipalloc-1deffb20)

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

server-ip-1 = 34.210.66.130
server-ip-2 = 52.38.3.67
Congratulations! You've set up two Rails-ready servers that are entirely documented through code, while keeping things DRY and easy to read. You're well on your way to having a production-ready Rails environment that's well documented, easy to modify and source controlled!

# Destroy it all
We're building infrastructure as code. It costs money to rent servers and we don't have any to spare. Let's destroy this infrastructure until the next chapter to save a little money.

$ terraform destroy
Get used to destroying resources you don't need! It's both fun and cost efficient. This is infrastructure as code, after all.

# Wrapping it up
In this chapter, we created a module rails-server and used it to instantiate two fully-provisioned servers. We utilized input variables to quickly modify module resources and output variables to grab the data we were interested in.

In the next chapter, we're going to expand this idea further by creating a running, production-ready Rails application environment through Terraform configurations.
