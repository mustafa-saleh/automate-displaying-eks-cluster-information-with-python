# 14 - Automation with Python

## 1 - Introduction to Boto Library (AWS SDK for Python)

Boto is the Amazon Web Services (AWS) Software Development Kit (SDK) for Python, which allows Python developers to write software that makes use of AWS services.

## 2 - Install Boto3 and connect to AWS

Install Boto3 using pip:

```bash
pip install boto3
```

## 3 - Getting familiar with Boto

Boto documentation: https://boto3.amazonaws.com/v1/documentation/api/latest/index.html or https://docs.aws.amazon.com/boto3/latest/

```py
import boto3

# Create an ec2 client
ec2 = boto3.client('ec2', region_name='us-west-2')

# create an ec2 resource
ec2_resource = boto3.resource('ec2', region_name='us-west-2')

new_vpc = ec2_resource.create_vpc(CidrBlock='10.0.0.0/16')

new_vpc.create_tags(Tags=[{"Key": "Name", "Value": "my-vpc"}])

new_vpc.create_subnet(CidrBlock='10.0.1.0/24')

new_vpc.create_subnet(CidrBlock='10.0.2.0/24')

# List all vpcs
response = ec2.describe_vpcs()
for vpc in response['Vpcs']:
    print(vpc['VpcId'])
```

Boto client vs resource:

- Client: 
  - Low-level service access, maps directly to AWS service APIs.
- Resource: 
  - Higher-level, object-oriented API, abstracts some of the low-level details.
  - Return resource objects that can be used for subsequent calls.

## 4 - Terraform vs Python - understand when to use which tool

Terraform is a declarative infrastructure as code tool, while Python (with Boto3) is an imperative programming language.
Terraform is idempotent & best suited for provisioning and managing infrastructure in a predictable and repeatable manner, while Python is better for automating tasks, integrating with other systems, and performing complex logic.

## 5 - Health Check: EC2 Status Checks

Create 3 instances with terraform

```tf
provider "aws" {
  region = "eu-west-3"
}

variable vpc_cidr_block {}
variable subnet_cidr_block {}
variable avail_zone {}
variable env_prefix {}
variable instance_type {}
variable my_ip {}
variable public_key_location {}

data "aws_ami" "amazon-linux-image" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

output "ami_id" {
  value = data.aws_ami.amazon-linux-image.id
}

resource "aws_vpc" "myapp-vpc" {
  cidr_block = var.vpc_cidr_block
  tags = {
      Name = "${var.env_prefix}-vpc"
  }
}

resource "aws_subnet" "myapp-subnet-1" {
  vpc_id = aws_vpc.myapp-vpc.id
  cidr_block = var.subnet_cidr_block
  availability_zone = var.avail_zone
  tags = {
      Name = "${var.env_prefix}-subnet-1"
  }
}

resource "aws_security_group" "myapp-sg" {
  name   = "myapp-sg"
  vpc_id = aws_vpc.myapp-vpc.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.my_ip]
  }

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    cidr_blocks     = ["0.0.0.0/0"]
    prefix_list_ids = []
  }

  tags = {
    Name = "${var.env_prefix}-sg"
  }
}

resource "aws_internet_gateway" "myapp-igw" {
	vpc_id = aws_vpc.myapp-vpc.id
    
    tags = {
     Name = "${var.env_prefix}-internet-gateway"
   }
}

resource "aws_route_table" "myapp-route-table" {
   vpc_id = aws_vpc.myapp-vpc.id

   route {
     cidr_block = "0.0.0.0/0"
     gateway_id = aws_internet_gateway.myapp-igw.id
   }

   # default route, mapping VPC CIDR block to "local", created implicitly and cannot be specified.

   tags = {
     Name = "${var.env_prefix}-route-table"
   }
 }

# Associate subnet with Route Table
resource "aws_route_table_association" "a-rtb-subnet" {
  subnet_id      = aws_subnet.myapp-subnet-1.id
  route_table_id = aws_route_table.myapp-route-table.id
}

resource "aws_key_pair" "ssh-key" {
  key_name   = "myapp-key"
  public_key = file(var.public_key_location)
}

output "server-ip" {
    value = aws_instance.myapp-server.public_ip
}

resource "aws_instance" "myapp-server" {
  ami                         = data.aws_ami.amazon-linux-image.id
  instance_type               = var.instance_type
  key_name                    = "myapp-key"
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.myapp-subnet-1.id
  vpc_security_group_ids      = [aws_security_group.myapp-sg.id]
  availability_zone			      = var.avail_zone

  tags = {
    Name = "${var.env_prefix}-server"
  }

  user_data = file("entry-script.sh")
  
  user_data_replace_on_change = true

}

resource "aws_instance" "myapp-server-two" {
  ami                         = data.aws_ami.amazon-linux-image.id
  instance_type               = var.instance_type
  key_name                    = "myapp-key"
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.myapp-subnet-1.id
  vpc_security_group_ids      = [aws_security_group.myapp-sg.id]
  availability_zone			      = var.avail_zone

  tags = {
    Name = "${var.env_prefix}-server-two"
  }

  user_data = file("entry-script.sh")
  
  user_data_replace_on_change = true

}

resource "aws_instance" "myapp-server-three" {
  ami                         = data.aws_ami.amazon-linux-image.id
  instance_type               = var.instance_type
  key_name                    = "myapp-key"
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.myapp-subnet-1.id
  vpc_security_group_ids      = [aws_security_group.myapp-sg.id]
  availability_zone			      = var.avail_zone

  tags = {
    Name = "${var.env_prefix}-server-three"
  }

  user_data = file("entry-script.sh")
  
  user_data_replace_on_change = true

}
```

Check the instance status with python

```py
import boto3
import schedule

ec2_client = boto3.client('ec2', region_name="eu-west-3")
ec2_resource = boto3.resource('ec2', region_name="eu-west-3")


def check_instance_status():
    statuses = ec2_client.describe_instance_status(
        IncludeAllInstances=True
    )
    for status in statuses['InstanceStatuses']:
        ins_status = status['InstanceStatus']['Status']
        sys_status = status['SystemStatus']['Status']
        state = status['InstanceState']['Name']
        print(f"Instance {status['InstanceId']} is {state} with instance status {ins_status} and system status {sys_status}")
    print("#############################\n")


schedule.every(5).minutes.do(check_instance_status)

while True:
    schedule.run_pending()
```

##  6 - Write a Scheduled Task in Python

pip install schedule

```bash
pip install schedule
```

## 7 - Configure Server: Add Environment Tags to EC2 Instances

If organization has AWS resources in different regions, we can use tags to identify the environment of each resource. For example, we can add a tag called "Environment" with values like "Development", "Staging", or "Production" to each EC2 instance.

Python boto3 code to add tags to EC2 instances:

```py
import boto3

ec2_client_paris = boto3.client('ec2', region_name="eu-west-3")
ec2_resource_paris = boto3.resource('ec2', region_name="eu-west-3")
ec2_client_frankfurt = boto3.client('ec2', region_name="eu-central-1")
ec2_resource_frankfurt = boto3.resource('ec2', region_name="eu-central-1")

instance_ids_paris = []
instance_ids_frankfurt = []

# fetch instances & iterate to get their ids
reservations_paris = ec2_client_paris.describe_instances()

for reservation in reservations_paris['Reservations']:
  for instance in reservation['Instances']:
    instance_ids_paris.append(instance['InstanceId'])

# add tags to instances
ec2_resource_paris.create_tags(
  Resources=instance_ids_paris,
  Tags=[
    {
      'Key': 'Environment',
      'Value': 'Development'
    }
  ]
)

# fetch instances & iterate to get their ids
reservations_frankfurt = ec2_client_frankfurt.describe_instances()

for reservation in reservations_frankfurt['Reservations']:
  for instance in reservation['Instances']:
    instance_ids_frankfurt.append(instance['InstanceId'])

# add tags to instances
ec2_resource_frankfurt.create_tags(
  Resources=instance_ids_frankfurt,
  Tags=[
    {
      'Key': 'Environment',
      'Value': 'Production'
    }
  ]
)
```

## 8 - EKS cluster information

Boto3 can be used to fetch information about an EKS cluster, such as its name, status, endpoint, and version. Here's an example code snippet:

```py
import boto3
eks_client = boto3.client('eks', region_name="eu-west-3")

clusters = eks_client.list_clusters()['clusters']

for cluster in clusters:
  response = eks_client.describe_cluster(name=cluster)
  cluster_info = response['cluster']

  print(f"Cluster Name: {cluster_info['name']}")
  print(f"Status: {cluster_info['status']}")
  print(f"Endpoint: {cluster_info['endpoint']}")
  print(f"Version: {cluster_info['version']}")
```


