# terraform

provider "aws" {
  region = "us-east-1"
}

# create VPC

resource "aws_vpc" "my_vpc" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"

  tags = {
    Name = "prod"
  }
}
#create internet gateway=> send traffic out to internet

resource "aws_internet_gateway" "my_gateway" {
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "prod"
  }
}
#create customroute table

resource "aws_route_table" "my_customroute" {
  vpc_id = aws_vpc.my_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.my_gateway.id
  }

  route {
    ipv6_cidr_block        = "::/0"
    egress_only_gateway_id = aws_egress_only_internet_gateway.example.id
  }

  tags = {
    Name = "prod"
  }
}
#create subnet

resource "aws_subnet" "my_subnet" {
  vpc_id     = aws_vpc.my_vpc.id
  cidr_block = "10.0.1.0/24"

  tags = {
    Name = "Main"
  }
}

#associate subnet with route table

resource "aws_route_table_association" "my_associate_subnet" {
  subnet_id      = aws_subnet.my_subnet.id
  route_table_id = aws_route_table.my_customroute.id
}
#create security group to allow port 22,80,443

resource "aws_security_group" "allow_web_traffic" {
  name        = "allow_web"
  description = "Allow TLS inbound traffic"
  vpc_id      = aws_vpc.my_vpc.id

  ingress {
    description      = "HTTPS from VPC"
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    cidr_blocks      = [0.0.0.0]
    
  }
  ingress {
    description      = "HTTP from VPC"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = [0.0.0.0/0]
    
  }
  ingress {
    description      = "SSH from VPC"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = [0.0.0.0/0]
    
  }



  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "allow_webtraffic"
  }
}
#create a network interface with an ip in the subnet that was created in step 4

resource "aws_network_interface" "my_networkinterface" {
  subnet_id       = aws_subnet.my_subnet.id
  private_ips     = ["10.0.1.50"]
  security_groups = aws_security_group.allow_web_traffic.id

}

#assign an elastic Ip to the network interface
resource "aws_eip" "my_elasticIP" {
  vpc                       = true
  network_interface         = aws_network_interface.my_networkinterface.id
  associate_with_private_ip = "10.0.1.50"
  depends_on = [aws_internet_gateway.my_gateway]
    
}
output "my_server_ip" {
  description = "Ip of publicip"
  value       = aws_eip.my_elasticIP.public_ip
}

#create ubuntu server and install/enable apache

resource "aws_instance" "my_instance" {
  ami           = "ami-tebyeyeami"
  instance_type = "t3.micro"
  key_name = "main_key"
  network_interface {
    network_interface_id = aws_network_interface.my_networkinterface.id
    device_index         = 0
  }

  tags = {
    Name = "prod"
  }
  user_data =   <<-EOF
      #!/bin/bash
      sudo apt update -y
      sudo apt install apache
      sudo apt start apache
      EOF
}


resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.main.id
  cidr_block = var.subnet_prifix

  tags = {
    Name = "Main"
  }
}

variable "subnet_prifix" {
  #type    = list(string)
  #default = ["us-west-1a"]
}

variable "subnet_prifix" {
  #type    = list(string)
  #default = ["us-west-1a"]
}

resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.main.id
  cidr_block = var.subnet_prifix[0]

  tags = {
    Name = "Main"
  }
}

resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.main.id
  cidr_block = var.subnet_prifix[1]

  tags = {
    Name = "Main"
  }
}

my test

