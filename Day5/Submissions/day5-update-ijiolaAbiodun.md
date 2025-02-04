# Day 5: Scaling Infrastructure

## Task Description

1. **Reading**: Completed Chapter 2 and started Chapter 3 on managing state.
   - Sections: "Deploying a load balancer, what is Terraform state, shared storage 
for state files, limitations with Terraform state."

2. **Udemy Videos**:
   - "Terraform data block" (Video 19)
   - "Terraform configuration block" (Video 20)
   - "Terraform module block" (Video 21)
   - "Terraform output block" (Video 22)

3. **Activity**: 
   - Scaled the web server cluster using Terraform to handle increased load.
   - Understood state files and how to manage them.

4. **Blog Post Ideas**:
   - "Managing high-traffic applications with AWS Elastic Load Balancer and 
Terraform"
   - "Best practices for managing state files in Terraform"

5. **Social Media Post**:
   - 🚀 Scaled my infrastructure with AWS Elastic Load Balancer today and using 
Terraform! #30daytfchallenge #HUG #hashicorp #HUGYDE @chiche #IaC

---

## Code Implementation

### `main.tf`

```hcl
region = var.aws_region
provider "aws" {
  region = var.aws_region
}

# Retrieve the list of AZs in the current AWS region
data "aws_availability_zones" "available" {}

# VPC
resource "aws_vpc" "vpc" {
  cidr_block = var.vpc_cidr

  tags = {
    Name        = var.vpc_name
    Environment = var.environment
    Terraform   = "true"
  }
}

# Private Subnets
resource "aws_subnet" "private_subnets" {
  for_each = var.private_subnets

  vpc_id            = aws_vpc.vpc.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, each.value)
  availability_zone = tolist(data.aws_availability_zones.available.names)[each.value 
- 1]

  tags = {
    Name      = each.key
    Terraform = "true"
  }
}

# Public Subnets
resource "aws_subnet" "public_subnets" {
  for_each = var.public_subnets

  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, each.value + 100)
  availability_zone       = 
tolist(data.aws_availability_zones.available.names)[each.value - 1]
  map_public_ip_on_launch = true

  tags = {
    Name      = each.key
    Terraform = "true"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "internet_gateway" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name      = "internet_gateway"
    Terraform = "true"
  }
}

# Security Group for Load Balancer
resource "aws_security_group" "lb_sg" {
  name        = "lb_security_group"
  description = "Allow inbound HTTP traffic to the Load Balancer"
  vpc_id      = aws_vpc.vpc.id

  ingress {
    description = "Allow HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = [var.variable_sub_cidr]
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "lb_sg"
  }
}

# Load Balancer
resource "aws_lb" "app_lb" {
  name               = "app-load-balancer"
  internal           = false
  load_balancer_type = "application"
  security_groups   = [aws_security_group.lb_sg.id]
  subnets            = [for subnet in aws_subnet.public_subnets : subnet.id]
  enable_deletion_protection = true

  tags = {
    Name = "app_lb"
  }
}

# Target Group for ALB
resource "aws_lb_target_group" "app_target_group" {
  name     = "app-target-group"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.vpc.id

  health_check {
    interval            = 30
    path                = "/"
    protocol            = "HTTP"
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }

  tags = {
    Name = "app_target_group"
  }
}

# Auto Scaling Launch Configuration
resource "aws_launch_configuration" "asg_launch_config" {
  name          = "asg-launch-config"
  image_id      = var.ami_id
  instance_type = "t2.micro"
  security_groups = [aws_security_group.lb_sg.id]

  lifecycle {
    create_before_destroy = true
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "asg" {
  desired_capacity     = var.asg_desired_capacity
  max_size             = var.asg_max_size
  min_size             = var.asg_min_size
  launch_configuration = aws_launch_configuration.asg_launch_config.id
  vpc_zone_identifier  = [for subnet in aws_subnet.private_subnets : subnet.id]
  target_group_arns    = [aws_lb_target_group.app_target_group.arn]

  tags = [
    {
      key                 = "Name"
      value               = "asg-instance"
      propagate_at_launch = true
    },
    {
      key                 = "Environment"
      value               = var.environment
      propagate_at_launch = true
    }
  ]
}


# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "asg_log_group" {
  name = "/aws/autoscaling/auto-scaling-group-logs"
}

# CloudWatch Alarm for CPU Utilization
resource "aws_cloudwatch_metric_alarm" "cpu_alarm" {
  alarm_name          = "high-cpu-utilization"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "1"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = "300"
  statistic           = "Average"
  threshold           = "80"
  alarm_description   = "Trigger when CPU utilization exceeds 80%"
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.asg.name
  }

  actions_enabled = true
  alarm_actions   = [aws_sns_topic.my_sns_topic.arn]
}

# SNS Topic for CloudWatch Alarm
resource "aws_sns_topic" "my_sns_topic" {
  name = "my-sns-topic"
}

# S3 Bucket and ACL (unchanged from your example)
resource "aws_s3_bucket" "my_new_s3_bucket" {
  bucket = "my-new-tf-s3-bucket-${random_id.randomness.hex}"

  tags = {
    Name    = "My S3 Bucket"
    Purpose = "Adding S3 Bucket"
  }
}

resource "aws_s3_bucket_acl" "my_new_s3_bucket_acl" {
  bucket = aws_s3_bucket.my_new_s3_bucket.id
  acl    = "private"
}

resource "aws_security_group" "my_new_security_group" {
  name        = "web_server_inbound"
  description = "Allow Inbound traffic on tcp/443"
  vpc_id      = aws_vpc.vpc.id

  ingress {
    description = "Allows 443 from the internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.variable_sub_cidr]
  }

  tags = {
    Name = "web_server_inbound"
  }
}
