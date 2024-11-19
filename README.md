# Project: Deploy Multiple EC2 Instances and Load Balancer on AWS Using Terraform

### Prerequisites

1. **AWS Account**: Ensure you have an active AWS account.
2. **SSH Key Pair**: Create an SSH key pair in your AWS account and download the `.pem` file.
3. **Amazon Linux Instance**: Ensure you have an Amazon Linux instance running with access to install and run Terraform.

### Step 1: Install Terraform on Amazon Linux

1. **Update Package Repository**:
   ```sh
   sudo yum update -y
   ```

2. **Download and Unzip Terraform**:
   ```sh
   wget https://releases.hashicorp.com/terraform/1.0.11/terraform_1.0.11_linux_amd64.zip
   unzip terraform_1.0.11_linux_amd64.zip -d /usr/local/bin
   ```

3. **Verify Terraform Installation**:
   ```sh
   terraform -version
   ```

### Step 2: Configure AWS CLI

1. **Install AWS CLI (if not already installed)**:
   ```sh
   sudo yum install aws-cli -y
   ```

2. **Configure AWS CLI**:
   ```sh
   aws configure
   ```
   - Enter your AWS Access Key ID, Secret Access Key, Default region name (`us-east-2`), and Default output format (e.g., `json`).

### Step 3: Create a Terraform Configuration File

1. **Create a Project Directory**:
   ```sh
   mkdir aws-vpc-app-lb
   cd aws-vpc-app-lb
   ```

2. **Create and Edit `main.tf`**:
   ```sh
   nano main.tf
   ```

3. **Add the Following Configuration to `main.tf`**:

```hcl
provider "aws" {
  region = "us-east-2"
}

# Create a VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "main-vpc"
  }
}

# Create Subnets in different Availability Zones
resource "aws_subnet" "subnet_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-2a"
  tags = {
    Name = "subnet-a"
  }
}

resource "aws_subnet" "subnet_b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-2b"
  tags = {
    Name = "subnet-b"
  }
}

# Create an Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "main-igw"
  }
}

# Create a Route Table
resource "aws_route_table" "rt" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = {
    Name = "main-route-table"
  }
}

# Associate Route Table with Subnets
resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.subnet_a.id
  route_table_id = aws_route_table.rt.id
}

resource "aws_route_table_association" "b" {
  subnet_id      = aws_subnet.subnet_b.id
  route_table_id = aws_route_table.rt.id
}

# Create a Security Group
resource "aws_security_group" "allow_http" {
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
    Name = "allow-http"
  }
}

# Create EC2 Instances
resource "aws_instance" "app_server_a" {
  ami           = "ami-0942ecd5d85baa812"  # Your AMI ID
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.subnet_a.id
  vpc_security_group_ids = [aws_security_group.allow_http.id]
  tags = {
    Name = "app-server-a"
  }
}

resource "aws_instance" "app_server_b" {
  ami           = "ami-0942ecd5d85baa812"  # Your AMI ID
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.subnet_b.id
  vpc_security_group_ids = [aws_security_group.allow_http.id]
  tags = {
    Name = "app-server-b"
  }
}

# Create a Load Balancer
resource "aws_lb" "app_lb" {
  name               = "app-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.allow_http.id]
  subnets            = [aws_subnet.subnet_a.id, aws_subnet.subnet_b.id]

  tags = {
    Name = "app-lb"
  }
}

# Create Target Group
resource "aws_lb_target_group" "app_tg" {
  name     = "app-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    interval            = 30
    path                = "/"
    port                = "80"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
    healthy_threshold   = 2
  }

  tags = {
    Name = "app-tg"
  }
}

# Create Load Balancer Listener
resource "aws_lb_listener" "app_lb_listener" {
  load_balancer_arn = aws_lb.app_lb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_tg.arn
  }

  tags = {
    Name = "app-lb-listener"
  }
}

# Register Targets
resource "aws_lb_target_group_attachment" "app_a" {
  target_group_arn = aws_lb_target_group.app_tg.arn
  target_id        = aws_instance.app_server_a.id
  port             = 80
}

resource "aws_lb_target_group_attachment" "app_b" {
  target_group_arn = aws_lb_target_group.app_tg.arn
  target_id        = aws_instance.app_server_b.id
  port             = 80
}
```

4. **Save and Exit**:
   - In Nano, press `Ctrl + O`, `Enter`, then `Ctrl + X` to save and exit the editor.

### Step 4: Initialize and Apply the Terraform Configuration

1. **Initialize Terraform**:
   ```sh
   terraform init
   ```

2. **Plan the Deployment**:
   ```sh
   terraform plan
   ```

3. **Apply the Configuration**:
   ```sh
   terraform apply
   ```
   - When prompted to confirm, type `yes` and press `Enter`.

### Step 5: Verify the Deployment

1. **Check AWS Management Console**:
   - Go to the **EC2 Dashboard** and **VPC Dashboard** in the AWS Management Console.
   - Verify that the VPC, subnets, Internet Gateway, Route Table, security group, EC2 instances, and load balancer have been created.

2. **Test Load Balancer**:
   - Navigate to the **Load Balancer** section in the AWS Management Console.
   - Find your load balancer and copy its **DNS name**.
   - Paste the DNS name into your web browser to verify that traffic is being distributed between your EC2 instances.

---

By following these steps, you'll be able to create a VPC, deploy two applications in different availability zones, and set up a load balancer to balance the load between the instances automatically. This guide provides a complete, step-by-step process to set up and deploy your infrastructure!!
