
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
