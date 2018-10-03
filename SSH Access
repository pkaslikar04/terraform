# SSH Access
In the spirit of this chapter, we're going to set up an SSH key pair through Terraform and the CLI. There is a web-based interface you can use to do this. But where's the fun in that?

The key pair will be used to SSH into the server we've set up, without a password. Instead it'll use a PEM file that we've set up. A private key that only our machine has access to.

# Note: Public key encryption is an extremely interesting and useful thing to know. If you're not familiar with it, I definitely recommend taking a look!

Generate the Public and Private Keys
Open up another terminal window, and execute the following:

$ cd ~/.ssh/
$ ssh-keygen -t rsa -b 2048 -v
For the filename, let's just enter my_test_key, so that the full location of the private key is ~/.ssh/my_test_key and the public key is located at ~/.ssh/my_test_key.pub. Let's leave the passwords blank.

Apply the Key Pair through Terraform
Back in the project directory (2-install-rails), execute the following command:

$ cd ~/.ssh/
$ ssh-keygen -t rsa -b 2048 -v
Alright! The last step is to tell AWS about this key pair through Terraform a small configuration.

# Create a file named key_pair.tf with the contents:

resource "aws_key_pair" "my-test-key" {
  key_name   = "test-key"
  public_key = "${file("my_test_key.pub")}"
}
view raw2-installing-rails-key-pair.tf hosted with ❤ by GitHub
That will set the Key Pair up within Amazon. Now we just have to use it!

# Using a Key Pair for SSH
In order to use the key pair we've created, we have to add the key name to the configuration of the EC2 instance.

# Open up the main.tf file and make the addition of a key_name attribute:

resource "aws_instance" "my-test-instance" {
  ami             = "${data.aws_ami.ubuntu.id}"
  instance_type   = "t2.micro"
  key_name        = "${aws_key_pair.my-test-key.key_name}"

  security_groups = [
    "${aws_security_group.allow_ssh.name}",
    "${aws_security_group.allow_outbound.name}"
  ]

  tags {
    Name = "test-instance"
  }
}

# This will attach the key pair we just generated to the instance and allow it to be used for SSH autentication.

# Connect via SSH
If you now run terraform apply, and pull the IP address out of the final output, you should be able to successfuly SSH into the machine using:

$ ssh 52.24.133.0 -l ubuntu -i ~/.ssh/my_test_key
We now have a fully working, fully SSH-able, Ubuntu server configuration!

# Let's get Rails on it!

Install Rails
After setting up the security groups and SSH keys, it's time to install Rails. We'll do this through a configuration called a provisioner.

Adding the provisioner
Modify main.tf to look like this:

resource "aws_instance" "my-test-instance" {
  ami             = "${data.aws_ami.ubuntu.id}"
  instance_type   = "t2.micro"
  key_name        = "${aws_key_pair.my-test-key.key_name}"

  security_groups = [
    "${aws_security_group.allow_ssh.name}",
    "${aws_security_group.allow_outbound.name}"
  ]

  provisioner "remote-exec" {
    inline = [
      "command curl -sSL https://rvm.io/mpapis.asc | gpg --import -",
      "\\curl -sSL https://get.rvm.io | bash -s stable --rails",
    ]

    connection {
      type          = "ssh"
      user          = "ubuntu"
      private_key   = "${file("~/.ssh/my_test_key")}"
    }
  }

  tags {
    Name = "test-instance"
  }
}

The provisioner we're using here is a remote-exec. When the server is created, this provisioner will connect to it with the details given in the connection block.
The provisioner will then execute the bash commands placed in the in the inline attribute. In this case, the server will download and install RVM along with the latest version of Rails.

# Recreating the server
For whatever reason, Terraform does not seem to force an update when the provisioner on an ec2_instance is modified. That being the case, we'll force the update ourselves!

This time, let's destroy only the server. Execute the following command to exclusively destroy the server and the things that depend on it (the Elastic IP, in this case):

$ terraform destroy -target="aws_instance.my-test-instance"
Now, you can execute terraform apply one more time to bring everything back online and install Rails!

Checking the installation
If all went well, you should receive output containing the following:

Outputs:

server-ip = 52.43.141.214
You can use that to SSH into the server and check if Rails is installed properly:

$ ssh 52.43.141.214 -l ubuntu -i ~/.ssh/my_test_key

ubuntu@ip-172-31-29-13:~$ rails -v
and get an output like:

Rails 5.1.4
Congratulations! You've done a ton of stuff in AWS and Terraform and are well on your way to having a fully-deployed application.
