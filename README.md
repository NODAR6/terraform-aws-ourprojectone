# OURPROJECTONE

# Project Overview
This project was created to design a fully functional AWS infrastructure for a WordPress deployment. The architecture includes a VPC, subnets, NAT and Internet gateways, autoscaling, and a load balancer, as well as an RDS database cluster and Route 53 configuration for DNS management.

![Screenshot](https://github.com/freyac777/terraform-aws-ourprojectone/assets/164959620/39dfd9a5-9349-4812-a976-ca0f91367833)

# Day 1: Setting up VPC and Subnets
I started by creating a VPC with three public and three private subnets. Additionally, I set up a NAT gateway, an internet gateway, and route tables to facilitate communication between the resources.

## VPC and Subnets Configuration
```hcl
# Create VPC
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr_block
}

# Create subnets
resource "aws_subnet" "public" {
  count                   = 3
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidr_blocks[count.index]
  map_public_ip_on_launch = true
  availability_zone       = element(var.azs, count.index)
}

resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidr_blocks[count.index]
  availability_zone = element(var.azs, count.index)
}

# Create Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

# Create NAT Gateways
resource "aws_nat_gateway" "ngw" {
  count         = 3
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
}

resource "aws_eip" "nat" {
  count = 3
}

# Create Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "public-route-table"
  }
}

resource "aws_route_table_association" "public" {
  count          = 3
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.ngw[count.index].id
  }

  tags = {
    Name = "private-route-table"
  }
}

resource "aws_route_table_association" "private" {
  count          = 3
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

## Variables Configuration
```hcl
variable "region" {
  description = "AWS region"
  default     = "us-east-1"
}

variable "vpc_cidr_block" {
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidr_blocks" {
  description = "CIDR blocks for the public subnets"
  default     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

variable "private_subnet_cidr_blocks" {
  description = "CIDR blocks for the private subnets"
  default     = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"]
}

variable "azs" {
  type        = list(string)
  description = "Availability Zones"
  default     = ["us-east-1a", "us-east-1b", "us-east-1c"]
}
```

# Day 2: Autoscaling, Templates, and Load Balancer
On the second day, I set up autoscaling, launch templates, and a load balancer. I also configured user data to deploy WordPress.

## Launch Template and Autoscaling
```hcl
resource "aws_launch_template" "projecttemplate" {
  name_prefix   = "projecttemplate-launch-template"
  image_id      = "ami-07caf09b362be10b8"
  instance_type = "t2.large"
  key_name      = "local"

  network_interfaces {
    security_groups           = [aws_security_group.projectsec.id]
    associate_public_ip_address = true
    subnet_id                 = aws_subnet.public[0].id
    delete_on_termination     = true
  }

  user_data = base64encode(<<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd php php-mysqlnd
    systemctl start httpd
    systemctl enable httpd
    wget -c https://wordpress.org/latest.tar.gz
    tar -xvzf latest.tar.gz -C /var/www/html
    cp -r /var/www/html/wordpress/* /var/www/html/
    chown -R apache:apache /var/www/html/
    mv /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
    sed -i "s/database_name_here/admin/" /var/www/html/wp-config.php
    sed -i "s/username_here/admin/" /var/www/html/wp-config.php
    sed -i "s/password_here/password/" /var/www/html/wp-config.php
    sed -i "s/localhost/${aws_db_instance.writer.endpoint}/" /var/www/html/wp-config.php
    EOF
  )
}

resource "aws_autoscaling_group" "asg" {
  name = "projecttemplate-asg"

  launch_template {
    id = aws_launch_template.projecttemplate.id
  }

  min_size                = 1
  max_size                = 5
  desired_capacity        = 1
  health_check_type       = "EC2"
  health_check_grace_period = 300
}
```

## Load Balancer
```hcl
resource "aws_lb" "wordpress_alb" {
  name               = "wordpress-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.projectsec.id]
  subnets            = aws_subnet.public[*].id
}

resource "aws_lb_listener" "wordpress" {
  load_balancer_arn = aws_lb.wordpress_alb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.Wordpress_TG.arn
  }
}

resource "aws_lb_target_group" "Wordpress_TG" {
  name     = "wordpress-target-group"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
}

resource "aws_autoscaling_attachment" "wordpress" {
  autoscaling_group_name = aws_autoscaling_group.asg.id
  lb_target_group_arn    = aws_lb_target_group.Wordpress_TG.arn
}
```

# Backend Configuration for Collaboration
```hcl
terraform {
  backend "s3" {
    bucket = "nodar-terraform6"
    key    = "terraform.tfstate"
    region = "us-east-1"
  }
}
```

# RDS Cluster Configuration
```hcl
resource "aws_db_instance" "writer" {
  engine         = "mysql"
  instance_class = "db.t3.medium"
  username       = "admin"
  password       = "password"
  allocated_storage = 20
}

resource "aws_db_instance" "reader" {
  engine         = "mysql"
  instance_class = "db.t3.medium"
  username       = "admin"
  password       = "password"
  allocated_storage = 20
}
```

# Route 53 Configuration
```hcl
resource "aws_route53_record" "wordpress" {
  zone_id = "Z00084981ESKE4O2GY2WC"
  name    = "wordpress"
  type    = "A"
  alias {
    name                   = aws_lb.wordpress_alb.dns_name
    zone_id                = aws_lb.wordpress_alb.zone_id
    evaluate_target_health = true
  }
}
