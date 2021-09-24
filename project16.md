# Architectural Diagram
- Project Architectural diagram - ![](imgs/architecture-diagram.png)
# Prerequisites
- Create an IAM user named **terraform**, ensure the user has only programmatic access and grant the AdmninistratorAccess permissions. ![](imgs/create-iam-user.png)
- Download the secret access key and access key ID CSV file.
- Install boto3 using pip
  ```
  pip install boto3
  ```
- Install AWS CLI and configure programmatic access using
  ```
  aws configure
  ```
  Paste in the access key ID and secret access key when prompted. Leave everything else as default.
- Create an S3 bucket to store Terraform state files.
  ![](imgs/bucket.png)
- Paste the following python code in a file and run to ensure programmatic access was successfully configured.
  ```py
  import boto3
  s3 = boto3.resource('s3')
  for bucket in s3.buckets.all():
    print(bucket.name)
  ```
    Test programmatic access
    ![](imgs/test-programmatic-bucket-access.png)
# Step 1: VPC, Subnets and Security Groups
## Step 1.1: Create a directory structure and set up Terraform CLI
- Create a folder named PBL and a file called main.tf
- Set up Terraform CLI
  ```
  $ sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl
  $ curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
  $ sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
  $ sudo apt-get update && sudo apt-get install terraform
  $ terraform -help
  ```
## Step 1.2: Create first VPC
- Add AWS as provider and a resource to create a VPC in the main.tf file
  ```
  provider "aws" {
  region = "eu-central-1"
  }

  # Create VPC
  resource "aws_vpc" "main" {
    cidr_block                     = "172.16.0.0/16"
    enable_dns_support             = "true"
    enable_dns_hostnames           = "true"
    enable_classiclink             = "false"
    enable_classiclink_dns_support = "false"
  }
  ```
- Download all necessary plugins for Terraform to run
  ```sh
  cd <where-main.tf-is-located>
  terraform init
  ```
  ![](imgs/tfinit.png)
  
- Run 
  ```sh
  terraform plan
  ```
  to see the changes that will be effected. If you are satisfied with everything.
  ![](imgs/plan1.png)

  - Run
  ```sh
  terraform apply
  ```
  to effect the changes. If you are satisfied with the output of terraform plan.
  ![](imgs/apply1.png)
## Step 1.3: Create subnet resource
- Add the following block of code to create two VPC subnets
  ```
    # Create public subnets1
    resource "aws_subnet" "public1" {
      vpc_id                     = aws_vpc.main.id
      cidr_block                 = "172.16.0.0/24"
      map_public_ip_on_launch    = true
      availability_zone          = "eu-central-1a"
    }

    # Create public subnet2
    resource "aws_subnet" "public2" {
      vpc_id                     = aws_vpc.main.id
      cidr_block                 = "172.16.1.0/24"
      map_public_ip_on_launch    = true
      availability_zone          = "eu-central-1b"
    }
  ```
- Run
  ```sh
  terraform plan
  terraform apply
  ```
  Type yes when prompted
  ![](imgs/apply2.png)
- Destroy the current infrastructure with
  ```sh
  terraform destroy
  ```
  Type yes when prompted
  ![](imgs/destroy1.png)

# Step 2: Fixing limitations(bad practices) by code refactoring
## Step 2.1: Fixing hard coded values
- Adding the following block to the main.tf file
  ```
    variable "region" {
        default = "eu-central-1"
    }

    variable "vpc_cidr" {
        default = "172.16.0.0/16"
    }

    variable "enable_dns_support" {
        default = "true"
    }

    variable "enable_dns_hostnames" {
        default ="true" 
    }

    variable "enable_classiclink" {
        default = "false"
    }

    variable "enable_classiclink_dns_support" {
        default = "false"
    }

    provider "aws" {
    region = var.region
    }

    # Create VPC
    resource "aws_vpc" "main" {
    cidr_block                     = var.vpc_cidr
    enable_dns_support             = var.enable_dns_support 
    enable_dns_hostnames           = var.enable_dns_support
    enable_classiclink             = var.enable_classiclink
    enable_classiclink_dns_support = var.enable_classiclink

    }
## Step 2.2: Fixing multiple resource blocks
- Use Terraform **data sources** to fetch list of availability zones from AWS
  ```
  # Get list of availability zones
  data "aws_availability_zones" "available" {
        state = "available"
  }
  ```
- To make use of the new data resource, add **count** to the subnet block
  ```
  # Create public subnet1
  resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = "172.16.1.0/24"
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]
        }
- Make cidr_block dynamic
  ```
    # Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]
    }
  ```
  * You can experiment with the cidrsubnet function in the terraform console by typing **terraform console** in the terminal
## Step 2.3: Remove hard coded count value
- Introduce the **length()** function which returns the length of a given list, map or string. Update the subnet resource block like so
  ```
  # Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = length(data.aws_availability_zones.available.names)
        vpc_id                  = aws_vpc.main.id
        cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]
    }
- Update the count argument with a condition
  ```
  # Create public subnets
  resource "aws_subnet" "public" {
    count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
    vpc_id = aws_vpc.main.id
    cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
    map_public_ip_on_launch = true
    availability_zone       = data.aws_availability_zones.available.names[count.index]
    }
    ```
    ![](imgs/plan3.png)
    * Try changing the *preferred_number_of_public_subnets* to null to see how the condition works(We see that 3 public subnets are created)
    ![](imgs/null.png)
# Step 3: Introducing variables.tf and terraform.tfvars
## Step 3.1: Restructure directory tree
- Create two files, one named variables.tf and the other terraform.tfvars
- Paste in the following block in the variables.tf file
  ```
  variable "region" {
      default = "eu-central-1"
  }

  variable "vpc_cidr" {
    default = "172.16.0.0/16"
  }

  variable "enable_dns_support" {
    default = "true"
  }

  variable "enable_dns_hostnames" {
    default ="true" 
  }

  variable "enable_classiclink" {
    default = "false"
  }

  variable "enable_classiclink_dns_support" {
    default = "false"
  }

  variable "preferred_number_of_public_subnets" {
      default = null
  }
  ```
  and the following in the terraform.tfvars file
  ```
  region = "eu-central-1"

  vpc_cidr = "172.16.0.0/16" 

  enable_dns_support = "true" 

  enable_dns_hostnames = "true"  

  enable_classiclink = "false" 

  enable_classiclink_dns_support = "false" 

  preferred_number_of_public_subnets = 2
  ```
  Your directory structure should look like this:
  ```sh
  .
  ├── main.tf
  ├── terraform.tfstate
  ├── terraform.tfstate.backup
  ├── terraform.tfvars
  └── variables.tf

  0 directories, 5 files
  ```
  Run **terraform plan** and ensure everything runs successfully.
  ```sh
  terraform plan
  ```
  ![](imgs/final.png)