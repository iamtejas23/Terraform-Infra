# Kops-Terraform-Infra

To create Kubernetes infrastructure on AWS using `kops`, `Terraform`, and Amazon Linux EC2 instances, you'll need a `main.tf` file for Terraform that sets up the necessary AWS infrastructure. Below is a basic example to get you started. This configuration will set up a VPC, subnets, and security groups. You might need to adjust it based on your specific requirements and environment.

```hcl
provider "aws" {
  region = "us-west-2"  # Change to your desired AWS region
}

# Create a VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  enable_dns_support = true
  enable_dns_hostnames = true
  tags = {
    Name = "kops-vpc"
  }
}

# Create a public subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-west-2a"  # Change to your preferred AZ
  map_public_ip_on_launch = true
  tags = {
    Name = "kops-public-subnet"
  }
}

# Create a private subnet
resource "aws_subnet" "private" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "us-west-2a"  # Change to your preferred AZ
  tags = {
    Name = "kops-private-subnet"
  }
}

# Create an internet gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "kops-igw"
  }
}

# Create a route table for public subnet
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = {
    Name = "kops-public-route-table"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Create security group
resource "aws_security_group" "k8s" {
  vpc_id = aws_vpc.main.id
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 6443
    to_port     = 6443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = {
    Name = "kops-security-group"
  }
}

# Create an EC2 instance for kops (example only, adjust instance type and count as needed)
resource "aws_instance" "kops" {
  ami           = "ami-0abcdef1234567890"  # Replace with a valid Amazon Linux AMI ID
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public.id
  security_groups = [aws_security_group.k8s.name]
  key_name       = "your-key-pair"  # Replace with your key pair name

  tags = {
    Name = "kops-instance"
  }
}

output "instance_id" {
  value = aws_instance.kops.id
}
```

### Steps to Use This Configuration:

1. **Install Terraform**: Ensure Terraform is installed on your machine.
2. **Configure AWS CLI**: Ensure the AWS CLI is configured with the appropriate credentials.
3. **Initialize Terraform**: Run `terraform init` to initialize the Terraform configuration.
4. **Plan the Deployment**: Run `terraform plan` to see what changes will be made.
5. **Apply the Configuration**: Run `terraform apply` to create the resources.
