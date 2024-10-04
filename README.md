---
# AWS Infrastructure with Terraform

This project configures and provisions an AWS infrastructure using Terraform. It sets up a Load Balancer (ELB), an Auto Scaling Group (ASG), EC2 instances, and the necessary security groups to allow HTTP traffic.

## Infrastructure Components

### 1. **AWS Provider**
   - Configures the AWS provider to use the region `ap-south-1` (Asia Pacific - Mumbai).
   
   ```hcl
   provider "aws" {
     region = "ap-south-1"
   }
   ```

### 2. **Availability Zones**
   - Retrieves the list of availability zones in the configured AWS region.
   
   ```hcl
   data "aws_availability_zones" "all" {}
   ```

### 3. **Security Groups**
   
   - **Security Group for ELB**
     - Allows inbound HTTP traffic and unrestricted outbound traffic for the Elastic Load Balancer (ELB).
     ```hcl
     resource "aws_security_group" "elb" {
       name = "akash-example-elb"
       egress {
         from_port   = 0
         to_port     = 0
         protocol    = "-1"
         cidr_blocks = ["0.0.0.0/0"]
       }
       ingress {
         from_port   = var.elb_port
         to_port     = var.elb_port
         protocol    = "tcp"
         cidr_blocks = ["0.0.0.0/0"]
       }
     }
     ```

   - **Security Group for EC2 Instances**
     - Allows inbound HTTP traffic on the specified server port.
     ```hcl
     resource "aws_security_group" "instance" {
       name = "akash-example-instance-elb"
       ingress {
         from_port   = var.server_port
         to_port     = var.server_port
         protocol    = "tcp"
         cidr_blocks = ["0.0.0.0/0"]
       }
     }
     ```

### 4. **Elastic Load Balancer (ELB)**
   - Creates an Application ELB to route traffic across the Auto Scaling Group (ASG). The ELB is attached to a health check and listens for HTTP requests on the specified ports.
   
   ```hcl
   resource "aws_elb" "example" {
     name               = "akash-elb-example"
     security_groups    = [aws_security_group.elb.id]
     availability_zones = data.aws_availability_zones.all.names

     health_check {
       target              = "HTTP:${var.server_port}/"
       interval            = 30
       timeout             = 3
       healthy_threshold   = 2
       unhealthy_threshold = 2
     }

     listener {
       lb_port           = var.elb_port
       lb_protocol       = "http"
       instance_port     = var.server_port
       instance_protocol = "http"
     }
   }
   ```

### 5. **Launch Configuration**
   - Defines the EC2 instances in the Auto Scaling Group. Each instance runs a simple HTTP server displaying a custom HTML page.
   
   ```hcl
   resource "aws_launch_configuration" "example" {
     name = "akash-example-launchconfig"
     image_id        = "ami-0620d12a9cf777c87"  # Ubuntu Server 18.04 LTS in ap-south-01
     instance_type   = "t2.medium"
     security_groups = [aws_security_group.instance.id]

     user_data = <<-EOF
                 #!/bin/bash
                 echo '<html><body><h1 style="font-size:50px;color:blue;">AKASH <br> <font style="color:red;"> Email: akashnawin18.ia@gmail.com <br> <font style="color:green;"> +91-9916818558 </h1> </body></html>' > index.html
                 nohup busybox httpd -f -p "${var.server_port}" &
                 EOF

     lifecycle {
       create_before_destroy = true
     }
   }
   ```

### 6. **Auto Scaling Group (ASG)**
   - Creates an Auto Scaling Group with a minimum size of 2 and a maximum size of 10. The ASG is connected to the ELB and EC2 instances are tagged with `AKASH-ASG-PROJECT`.
   
   ```hcl
   resource "aws_autoscaling_group" "example" {
     name = "akash-example-asg"
     launch_configuration = aws_launch_configuration.example.id
     availability_zones   = data.aws_availability_zones.all.names
     min_size = 2
     max_size = 10
     load_balancers    = [aws_elb.example.name]
     health_check_type = "ELB"

     tag {
       key                 = "Name"
       value               = "AKASH-ASG-PROJECT"
       propagate_at_launch = true
     }
   }
   ```

## Variables

The following variables are configurable:

- **server_port**: The port the server will use for HTTP requests (default: 8080).
  
  ```hcl
  variable "server_port" {
    description = "The port the server will use for HTTP requests"
    type        = number
    default     = 8080
  }
  ```

- **elb_port**: The port the ELB will use for HTTP requests (default: 80).

  ```hcl
  variable "elb_port" {
    description = "The port the ELB will use for HTTP requests"
    type        = number
    default     = 80
  }
  ```

## Outputs

- **clb_dns_name**: Outputs the domain name of the load balancer.

  ```hcl
  output "clb_dns_name" {
    value       = aws_elb.example.dns_name
    description = "The domain name of the load balancer"
  }
  ```

## Requirements

- AWS CLI configured with sufficient permissions.
- Terraform installed on your local machine.
  
## Usage

1. Initialize Terraform:
   ```bash
   terraform init
   ```

2. Preview the changes Terraform will make:
   ```bash
   terraform plan
   ```

3. Apply the configuration:
   ```bash
   terraform apply
   ```

This setup creates an auto-scaling infrastructure with an ELB routing traffic to EC2 instances running a basic HTTP server displaying a custom HTML page with contact details.

---
