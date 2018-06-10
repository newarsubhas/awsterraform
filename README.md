
# Installation

Open Terminal by pressing command+space then type terminal and hit Enter key.
Install homebrew first.
```shell
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)" < /dev/null 2> /dev/null
```
Install terraform.
```shell
brew install terraform
```

# Building Infrastructure
After setting up the binaries we create an empty directory that will hold the new project. First thing we do is tell terraform which provider we are going to use. Since we are building Amazon AWS infrastructure we create a .tfvars file with our AWS IAM API credentials. For example, provider-credentials.tfvars with the following content:
```shell

cat > provider-credentials.tfvars <<EOF
provider = {
	access_key = "<AWS_ACCESS_KEY>"
	secret_key = "<AWS_SECRET_KEY>"
	region = "ap-southeast-2"
}
EOF

```
Then we create a .tf file where we create our first resource called provider. Lets name the file provider-config.tf and put the following content:
```shell
cat > provider-config.tf <<EOF
provider "aws" {
    access_key = "${var.provider["access_key"]}"
    secret_key = "${var.provider["secret_key"]}"
    region     = "${var.provider["region"]}"
}
EOF

```
Then we create a .tf file vpc_environment.tf where we put all essential variables needed to build the VPC, like VPC CIDR, AWS zone and regions, default EC2 instance type and the ssh key and other AWS related parameters:

```shell
cat > vpc_environment.tf <<EOF
/*=== VARIABLES ===*/
variable "provider" {
    type = "map"
    default = {
        access_key = "unknown"
        secret_key = "unknown"
        region     = "unknown"
    }
}

variable "vpc" {
    type    = "map"
    default = {
        "tag"         = "unknown"
        "cidr_block"  = "unknown"
        "subnet_bits" = "unknown"
        "owner_id"    = "unknown"
        "sns_topic"   = "unknown"
    }
}

variable "azs" {
    type = "map"
    default = {
        "ap-southeast-2" = "ap-southeast-2a,ap-southeast-2b,ap-southeast-2c"
        "eu-west-1"      = "eu-west-1a,eu-west-1b,eu-west-1c"
        "us-west-1"      = "us-west-1b,us-west-1c"
        "us-west-2"      = "us-west-2a,us-west-2b,us-west-2c"
        "us-east-1"      = "us-east-1c,us-east-1d,us-east-1e"
    }
}

variable "instance_type" {
    default = "t1.micro"
}

variable "key_name" {
    default = "unknown"
}

variable "nat" {
    type    = "map"
    default = {
        ami_image         = "unknown"
        instance_type     = "unknown"
        availability_zone = "unknown"
        key_name          = "unknown"
        filename          = "userdata_nat_asg.sh"
    }
}

/* Ubuntu Trusty 14.04 LTS (x64) */
variable "images" {
    type    = "map"
    default = {
        eu-west-1      = "ami-47a23a30"
        ap-southeast-2 = "ami-6c14310f"
        us-east-1      = "ami-2d39803a"
        us-west-1      = "ami-48db9d28"
        us-west-2      = "ami-d732f0b7"
    }
}

variable "env_domain" {
    type    = "map"
    default = {
        name    = "unknown"
        zone_id = "unknown"
    }
}
EOF

```
I have created most of the variables as generic and then passing on their values via separate .tfvars file vpc_environment.tfvars:
```shell
cat > vpc_environment.tfvars <<EOF
vpc = {
    tag                   = "TFTEST"
    owner_id              = "<owner-id>"
    cidr_block            = "10.99.0.0/20"
    subnet_bits           = "4"
    sns_email             = "<sns-email>"
}
key_name                  = "<ssh-key>"
nat.instance_type         = "m3.medium"
env_domain = {
    name                  = "mydomain.com"
    zone_id               = "<zone-id>"
}
EOF
```
We want our VPC to be created in a region with 3 AZ’s (Availability Zones) so we can spread our future instance nicely for HA. We create a new .tf file vpc.tf:
```shell
nano vpc.tf

/*=== VPC AND SUBNETS ===*/
resource "aws_vpc" "environment" {
    cidr_block           = "${var.vpc["cidr_block"]}"
    enable_dns_support   = true
    enable_dns_hostnames = true 
    tags {
        Name        = "VPC-${var.vpc["tag"]}"
        Environment = "${lower(var.vpc["tag"])}"
    }
}

resource "aws_internet_gateway" "environment" {
    vpc_id = "${aws_vpc.environment.id}"
    tags {
        Name        = "${var.vpc["tag"]}-internet-gateway"
        Environment = "${lower(var.vpc["tag"])}"
    }
}

resource "aws_subnet" "public-subnets" {
    vpc_id            = "${aws_vpc.environment.id}"
    count             = "${length(split(",", lookup(var.azs, var.provider["region"])))}"
    cidr_block        = "${cidrsubnet(var.vpc["cidr_block"], var.vpc["subnet_bits"], count.index)}"
    availability_zone = "${element(split(",", lookup(var.azs, var.provider["region"])), count.index)}"
    tags {
        Name          = "${var.vpc["tag"]}-public-subnet-${count.index}"
        Environment   = "${lower(var.vpc["tag"])}"
    }
    map_public_ip_on_launch = true
}

resource "aws_subnet" "private-subnets" {
    vpc_id            = "${aws_vpc.environment.id}"
    count             = "${length(split(",", lookup(var.azs, var.provider["region"])))}"
    cidr_block        = "${cidrsubnet(var.vpc["cidr_block"], var.vpc["subnet_bits"], count.index + length(split(",", lookup(var.azs, var.provider["region"]))))}"
    availability_zone = "${element(split(",", lookup(var.azs, var.provider["region"])), count.index)}"
    tags {
        Name          = "${var.vpc["tag"]}-private-subnet-${count.index}"
        Environment   = "${lower(var.vpc["tag"])}"
        Network       = "private"
    }
}

resource "aws_subnet" "private-subnets-2" {
    vpc_id            = "${aws_vpc.environment.id}"
    count             = "${length(split(",", lookup(var.azs, var.provider["region"])))}"
    cidr_block        = "${cidrsubnet(var.vpc["cidr_block"], var.vpc["subnet_bits"], count.index + (2 * length(split(",", lookup(var.azs, var.provider["region"])))))}"
    availability_zone = "${element(split(",", lookup(var.azs, var.provider["region"])), count.index)}"
    tags {
        Name          = "${var.vpc["tag"]}-private-subnet-2-${count.index}"
        Environment   = "${lower(var.vpc["tag"])}"
        Network       = "private"
    }
}

```
This will create a VPC for us with 3 sets of subnets, 2 private and 1 public (meaning will have the IGW as default gateway). For the private subnets we need to create a NAT instance to be used as internet gateway. We can create a new .tf file vpc_nat_instance.tf lets say where we create the resource:

```shell
nano vpc_nat_instance.tf

/*== NAT INSTANCE IAM PROFILE ==*/
resource "aws_iam_instance_profile" "nat" {
    name  = "${var.vpc["tag"]}-nat-profile"
    roles = ["${aws_iam_role.nat.name}"]
}

resource "aws_iam_role" "nat" {
    name = "${var.vpc["tag"]}-nat-role"
    path = "/"
    assume_role_policy = <<EOF
{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Action": "sts:AssumeRole",
            "Principal": {"AWS": "*"},
            "Effect": "Allow",
            "Sid": ""
        }
    ]
}
EOF
}

resource "aws_iam_policy" "nat" {
    name = "${var.vpc["tag"]}-nat-policy"
    path = "/"
    description = "NAT IAM policy"
    policy = <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:ModifyInstanceAttribute",
                "ec2:DescribeSubnets",
                "ec2:DescribeRouteTables",
                "ec2:CreateRoute",
                "ec2:ReplaceRoute"
            ],
            "Resource": "*"
        }
    ]
}
EOF
}

resource "aws_iam_policy_attachment" "nat" {
    name       = "${var.vpc["tag"]}-nat-attachment"
    roles      = ["${aws_iam_role.nat.name}"]
    policy_arn = "${aws_iam_policy.nat.arn}"
}

/*=== NAT INSTANCE ASG ===*/
resource "aws_autoscaling_group" "nat" {
    name                      = "${var.vpc["tag"]}-nat-asg"
    availability_zones        = "${split(",", lookup(var.azs, var.provider["region"]))}"
    vpc_zone_identifier       = ["${aws_subnet.public-subnets.*.id}"]
    max_size                  = 1
    min_size                  = 1
    health_check_grace_period = 60
    default_cooldown          = 60
    health_check_type         = "EC2"
    desired_capacity          = 1
    force_delete              = true
    launch_configuration      = "${aws_launch_configuration.nat.name}"
    tag {
      key                 = "Name"
      value               = "NAT-${var.vpc["tag"]}"
      propagate_at_launch = true
    }
    tag {
      key                 = "Environment"
      value               = "${lower(var.vpc["tag"])}"
      propagate_at_launch = true
    }
    tag {
      key                 = "Type"
      value               = "nat"
      propagate_at_launch = true
    }
    tag {
      key                 = "Role"
      value               = "bastion"
      propagate_at_launch = true
    }
    lifecycle {
      create_before_destroy = true
    }
}

resource "aws_launch_configuration" "nat" {
    name_prefix                 = "${var.vpc["tag"]}-nat-lc-"
    image_id                    = "${lookup(var.images, var.provider["region"])}"
    instance_type               = "${var.nat["instance_type"]}"
    iam_instance_profile        = "${aws_iam_instance_profile.nat.name}"
    key_name                    = "${var.key_name}"
    security_groups             = ["${aws_security_group.nat.id}"]
    associate_public_ip_address = true
    user_data                   = "${data.template_file.nat.rendered}"
    lifecycle {
      create_before_destroy = true
    }
}

data "template_file" "nat" {
    template = "${file("${var.nat["filename"]}")}"
    vars {
        cidr = "${var.vpc["cidr_block"]}"
    }
}
```
