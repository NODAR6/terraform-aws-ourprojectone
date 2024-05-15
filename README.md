
# OURPROJECTONE

# This is design of our project.

<img width="487" alt="Screenshot 2024-05-14 at 7 02 57 PM" src="https://github.com/freyac777/terraform-aws-ourprojectone/assets/164959620/39dfd9a5-9349-4812-a976-ca0f91367833">


# In our first day we start working on vpc and subnets, we cretaed vpc with three public and three private subnest.
# We also created NAT gateway and internet gateway for our subnets. We also did configuration of our route tables.
1. vpc name - main
2.  we define subnets public and private, Give them variables.
Here are code how this looks like:


```
# Crete vpc

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr_block
}

# Create subnets

resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidr_blocks[count.index]
  map_public_ip_on_launch = true
  availability_zone = element(var.azs, count.index)
}

resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidr_blocks[count.index]
  availability_zone = element(var.azs, count.index)
}


# Cretae internet Gateway

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}


# Create NAT Gateways


resource "aws_nat_gateway" "ngw" {
  count           = 3
  allocation_id   = aws_eip.nat[count.index].id
  subnet_id       = aws_subnet.private[count.index].id
}

resource "aws_eip" "nat" {
  count = 3
}


#Create Route tables

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
  count          = 3
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.ngw[count.index].id
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


# Also we create variables to get our job little bit easy.


```
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


# In next day we create autoscaling, Template and load balancer for our project, we create user data too. 
# And make sure that all this services was working good.
# We define our instances max and min capacity.
# alos we configure load balancer lisssner to port 80
# And in same file we attachedd autoscaling ion our wordpress


```

resource "aws_launch_template" "projecttemplate" {
  name_prefix   = "projecttemplate-launch-template"
  image_id      = "ami-07caf09b362be10b8" 
  instance_type = "t2.large"   
  key_name      = "local"   
  count = 1
  network_interfaces {
  security_groups = [aws_security_group.projectsec.id, aws_security_group.projectsec1.id]
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.public[count.index].id
  delete_on_termination       = true 
}



#   user_data = base64encode (<<EOF
# #!/bin/bash
# yum update -y
# yum install -y httpd php php-mysqlnd
# systemctl start httpd
# systemctl enable httpd
# wget -c https://wordpress.org/latest.tar.gz
# tar -xvzf latest.tar.gz -C /var/www/html
# cp -r /var/www/html/wordpress/* /var/www/html/
# chown -R apache:apache /var/www/html/

# cd /var/www/html/
# echo "
# <?php
# define( 'DB_NAME', 'admin' );
# define( 'DB_USER', 'admin' );
# define( 'DB_PASSWORD', 'password' );
# define( 'DB_HOST', 'terraform-**************************.ct6kq4048kie.us-east-1.rds.amazonaws.com' );
# define( 'DB_CHARSET', 'utf8mb4' );
# define( 'DB_COLLATE', '' );
# define( 'AUTH_KEY',         'admin' );                                                             
# define( 'SECURE_AUTH_SALT', 'admin' );
# define( 'LOGGED_IN_SALT',   'admin' );
# define( 'NONCE_SALT',       'admin' );
# \$table_prefix = 'wp_';
# define( 'WP_DEBUG', false );
# if ( ! defined( 'ABSPATH' ) ) {
#         define( 'ABSPATH', __DIR__ . '/' );
# }
# require_once ABSPATH . 'wp-settings.php';
# " > wp-config.php

# service httpd restart
# EOF
#   )
    
#}

user_data     = base64encode (<<-EOF
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

 
# Create auto scaling



resource "aws_autoscaling_group" "asg" {
  name = "projecttemplate-asg"

  launch_template {
    id = aws_launch_template.projecttemplate[0].id
    # vesrion = "$Latest"
  }

  min_size             = 1
  max_size             = 5
  desired_capacity     = 1  
  health_check_type    = "EC2"
  health_check_grace_period = 300  
  
}

# # Create an ALB


resource "aws_lb" "wordpress_alb" {
  name               = "wordpress-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.projectsec1.id]       # needt to change
  #subnets            = concat(aws_subnet.public[*].id, aws_subnet.private[*].id)
  subnets            = concat(aws_subnet.public[*].id)
  tags = {
    Name = "WordPressALB"
  }
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
   name     = "learn-asg-terramino"
   port     = 80
   protocol = "HTTP"
   vpc_id   = aws_vpc.main.id
 }

resource "aws_autoscaling_attachment" "wordpress_AAA" {
  autoscaling_group_name = aws_autoscaling_group.asg.id
  lb_target_group_arn  = aws_lb_target_group.Wordpress_TG.arn
  
}
```

# Our next step is security group. We create sec. groups for our subnets and database.

# alos we cretae backend and store our tfstate file in s3 bucket. Becouse in such case its easy for group members to work in same project at same time.

file for that: 

```
 terraform {
   backend "s3" {
     bucket = "nodar-terraform6"
     key    = "terraform.tfstate"
     region = "us-east-1"
   }
 }
```

# After that we did rds cluster and, we create db instacnes for writer and reader.  We choose our engine my mysql and
# also configure some instance classes with username and passwords.


## finnaly we create 53 to make sure that our project was running.
## We define here our  Zone id and domain name.


```
resource "aws_route53_record" "test" {
  zone_id = "Z00084981ESKE4O2GY2WC"  # Specify the Route 53 hosted zone ID where you want to create the record
  name    = "wordpress"  # Specify the domain name you want to associate with the ALB
  type    = "A"
  alias {
    name                   = aws_lb.wordpress_alb.dns_name  # Specify the DNS name of your ALB
    zone_id                = aws_lb.wordpress_alb.zone_id  # Specify the hosted zone ID of your ALB
    evaluate_target_health = true
    
  }
}
```
