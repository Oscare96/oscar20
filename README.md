terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"

    }
  }
}

provider "aws" {
  region                   = "us-east-1"
  shared_credentials_files = ["~/.aws/credentials"]
  profile                  = "vscode"
}

resource "aws_vpc" "mtn_vpc" {
  cidr_block           = "10.123.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true



  tags = {
    Name = "dev"
  }
}

resource "aws_subnet" "mtn_public_subnet" {
  vpc_id                  = aws_vpc.mtn_vpc.id
  cidr_block              = "10.123.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "us-east-1b"

  tags = {
    Name = "dev-public"
  }

}

resource "aws_internet_gateway" "mtn_internet_gateway" {
  vpc_id = aws_vpc.mtn_vpc.id

  tags = {
    Name = "dev-public"
  }

}

resource "aws_route_table" "mtn_public_rt" {
  vpc_id = aws_vpc.mtn_vpc.id

  tags = {
    Name = "dev-public"
  }

}

resource "aws_route" "default_route" {
  route_table_id         = aws_route_table.mtn_public_rt.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.mtn_internet_gateway.id

}

resource "aws_route_table_association" "mtn_public_assoc" {
  subnet_id      = aws_subnet.mtn_public_subnet.id
  route_table_id = aws_route_table.mtn_public_rt.id

}

resource "aws_security_group" "mtn_sg" {
  name        = "dev_sg"
  description = "dev security group"
  vpc_id      = aws_vpc.mtn_vpc.id


  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_key_pair" "mtn_key" {
    key_name = "mtnkey"
    public_key = file("~/.ssh/mtnkey.pub")
  
}

resource "aws_instance" "mtn_inst" {
    instance_type = "t2.micro"
    ami = data.aws_ami.server_ami.id

    tags = {
        Name = "dev.inst"
    }
     key_name = aws_key_pair.mtn_key.id
     vpc_security_group_ids = [aws_security_group.mtn_sg.id]
     subnet_id = aws_subnet.mtn_public_subnet.id
     user_data = file("user_data.tpl")

     root_block_device {
       volume_size = 10
     }
    provisioner "local-exec" {
        command = templatefile("${var.host_os}-ssh-config.tpl", {
            hostname = self.public_ip,
            user = "ubuntu",
            identityfile = "~/.ssh/mtnkey"

        })
        interpreter = ["bash", "-c"]
    }   
     }
