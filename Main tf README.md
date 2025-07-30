provider "aws" {
  region = var.aws_region
}

# VPC
resource "aws_vpc" "neo4j_vpc" {
  cidr_block = var.vpc_cidr
  tags = { Name = "neo4j-vpc" }
}

# Subnet
resource "aws_subnet" "neo4j_subnet" {
  vpc_id     = aws_vpc.neo4j_vpc.id
  cidr_block = var.subnet_cidr
  availability_zone = var.availability_zone
  tags = { Name = "neo4j-subnet" }
}

# Internet Gateway for public subnet
resource "aws_internet_gateway" "neo4j_gw" {
  vpc_id = aws_vpc.neo4j_vpc.id
  tags = { Name = "neo4j-gateway" }
}

resource "aws_route_table" "neo4j_rt" {
  vpc_id = aws_vpc.neo4j_vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.neo4j_gw.id
  }
  tags = { Name = "neo4j-rt" }
}

resource "aws_route_table_association" "neo4j_rta" {
  subnet_id      = aws_subnet.neo4j_subnet.id
  route_table_id = aws_route_table.neo4j_rt.id
}

# Security Group
resource "aws_security_group" "neo4j_sg" {
  name        = "neo4j-sg"
  description = "Allow SSH and Neo4j ports"
  vpc_id      = aws_vpc.neo4j_vpc.id

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Neo4j Bolt"
    from_port   = 7687
    to_port     = 7687
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Neo4j Browser"
    from_port   = 7474
    to_port     = 7474
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# EC2 Instance with user_data to download from JFrog
resource "aws_instance" "neo4j_ec2" {
  ami                         = var.ami_id
  instance_type               = var.instance_type
  subnet_id                   = aws_subnet.neo4j_subnet.id
  vpc_security_group_ids      = [aws_security_group.neo4j_sg.id]
  associate_public_ip_address = true
  key_name                    = var.key_name

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              amazon-linux-extras install java-openjdk11 -y
              mkdir -p /opt/neo4j
              cd /opt/neo4j

              # Download Neo4j and Corretto from Artifactory
              curl -u ${var.jfrog_user}:${var.jfrog_password} -O "${var.jfrog_url}/neo4j-enterprise.tar.gz"
              curl -u ${var.jfrog_user}:${var.jfrog_password} -O "${var.jfrog_url}/amazon-corretto.rpm"
              rpm -i amazon-corretto.rpm
              tar -xvzf neo4j-enterprise.tar.gz
              nohup ./neo4j-enterprise*/bin/neo4j start &
              EOF

  tags = {
    Name = "Neo4j-EC2"
  }
}
variables
variable "aws_region" {
  default = "us-east-1"
}

variable "vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "subnet_cidr" {
  default = "10.0.1.0/24"
}

variable "availability_zone" {
  default = "us-east-1a"
}

variable "ami_id" {
  description = "Amazon Linux 2 AMI ID"
  default     = "ami-0c2b8ca1dad447f8a"  # Update if needed
}

variable "instance_type" {
  default = "t3.medium"
}

variable "key_name" {
  description = "EC2 key pair name"
  type        = string
}

variable "jfrog_url" {
  description = "Base URL for JFrog Artifactory"
  type        = string
}

variable "jfrog_user" {
  description = "JFrog Artifactory username"
  type        = string
  sensitive   = true
}

variable "jfrog_password" {
  description = "JFrog Artifactory password or token"
  type        = string
  sensitive   = true
}
