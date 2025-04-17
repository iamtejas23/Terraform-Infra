``Terraform-Infra for app-servers``

```hcl
# Specify the AWS provider
provider "aws" {
  region = "us-east-1"  # Change to your preferred region
}

# Create a VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "my-vpc"
  }
}

# Create an internet gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "my-internet-gateway"
  }
}

# Create public subnets in two different Availability Zones
resource "aws_subnet" "public_1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  map_public_ip_on_launch = true
  tags = {
    Name = "my-public-subnet-1"
  }
}

resource "aws_subnet" "public_2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"
  map_public_ip_on_launch = true
  tags = {
    Name = "my-public-subnet-2"
  }
}

# Create a route table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "my-public-route-table"
  }
}

# Associate the route table with the public subnets
resource "aws_route_table_association" "public_1" {
  subnet_id      = aws_subnet.public_1.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public_2" {
  subnet_id      = aws_subnet.public_2.id
  route_table_id = aws_route_table.public.id
}

# Create a security group
resource "aws_security_group" "web-sg" {
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "web-security-group"
  }
}

# Create a launch template
resource "aws_launch_template" "web" {
  name_prefix   = "web-template"
  image_id      = "ami-07a6f770277670015"  # Amazon Linux AMI ID as per your request
  instance_type = "t3.micro"

  user_data = base64encode(<<-EOF
              #!/bin/bash
              amazon-linux-extras install -y docker
              service docker start
              usermod -a -G docker ec2-user
              docker run -d -p 80:3000 iamtejas23/zomato-clone:latest
              EOF
  )

  network_interfaces {
    associate_public_ip_address = true
    security_groups             = [aws_security_group.web-sg.id]
  }

  tags = {
    Name = "web-instance"
  }
}

# Create an autoscaling group
resource "aws_autoscaling_group" "web" {
  vpc_zone_identifier         = [aws_subnet.public_1.id, aws_subnet.public_2.id]  # Use both subnets
  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  min_size           = 1
  max_size           = 3
  desired_capacity   = 2
  health_check_type  = "EC2"
  health_check_grace_period = 300

  tag {
    key                 = "Name"
    value               = "web-server"
    propagate_at_launch = true
  }
}

# Create a load balancer
resource "aws_lb" "web" {
  name               = "web-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.web-sg.id]
  subnets            = [aws_subnet.public_1.id, aws_subnet.public_2.id]  # Use both subnets

  tags = {
    Name = "web-lb"
  }
}

# Create a target group
resource "aws_lb_target_group" "web" {
  name        = "web-targets"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "instance"

  health_check {
    path                = "/"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }
}

# Create a load balancer listener
resource "aws_lb_listener" "web" {
  load_balancer_arn = aws_lb.web.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

# Attach the autoscaling group to the target group
resource "aws_autoscaling_attachment" "asg_attachment" {
  autoscaling_group_name = aws_autoscaling_group.web.name
  lb_target_group_arn   = aws_lb_target_group.web.arn
  depends_on = [
    aws_autoscaling_group.web,
    aws_lb_target_group.web
  ]
}

# Output the DNS name of the load balancer
output "load_balancer_dns" {
  value = aws_lb.web.dns_name
}

```

### Steps to Use This Configuration:

1. **Install Terraform**: Ensure Terraform is installed on your machine.
2. **Configure AWS CLI**: Ensure the AWS CLI is configured with the appropriate credentials.
3. **Initialize Terraform**: Run `terraform init` to initialize the Terraform configuration.
4. **Plan the Deployment**: Run `terraform plan` to see what changes will be made.
5. **Apply the Configuration**: Run `terraform apply` to create the resources.
