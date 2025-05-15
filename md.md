Got it! You want to reuse an existing VPC, subnets, and other parameters as per the provided `tfvars`. I’ll rewrite the Terraform scripts to accommodate this.

Here is an updated version of the Terraform configuration that will utilize the existing VPC, subnets, and other parameters.

---

### **Directory Structure**:

```
/terraform
├── main.tf
├── variables.tf
├── terraform.tfvars
└── outputs.tf
```

---

### **`main.tf`**: Main configuration file for resources

```hcl
provider "aws" {
  region = var.aws_region
}

# MongoDB EC2 Instance Security Group (restricting access to specified CIDR blocks)
resource "aws_security_group" "mongo_sg" {
  name        = "mongo_security_group"
  description = "Security group for MongoDB with restricted access"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 27017
    to_port     = 27017
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidr_blocks  # Allow traffic only from specified CIDR blocks
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# MongoDB EC2 Instance
resource "aws_instance" "mongo_instance" {
  ami           = "ami-0c55b159cbfafe1f0"  # Use the latest Ubuntu AMI ID for your region (make sure it's correct)
  instance_type = var.instance_class
  subnet_id     = element(var.subnet_ids, 0)  # Use the first subnet from the provided list
  security_group_ids = [aws_security_group.mongo_sg.id]
  key_name      = var.key_name  # Make sure to set this variable to your SSH key pair name

  # User data to install MongoDB on EC2 instance
  user_data = <<-EOT
              #!/bin/bash
              apt-get update
              apt-get install -y mongodb
              systemctl start mongodb
              systemctl enable mongodb
              EOT

  tags = {
    Name = var.db_name
  }

  # Disable public IP for private access
  associate_public_ip_address = false

  # Wait for the instance to be ready before continuing
  wait_for_maintenance = true
}

# EBS Volume for MongoDB Storage
resource "aws_ebs_volume" "mongo_storage" {
  availability_zone = element(var.subnet_ids, 0) # Attach to the same availability zone as the EC2 instance
  size              = var.allocated_storage
  tags = {
    Name = "${var.db_name}-volume"
  }
}

# Attach the EBS volume to the MongoDB EC2 instance
resource "aws_volume_attachment" "mongo_volume_attachment" {
  device_name = "/dev/sdh"
  instance_id = aws_instance.mongo_instance.id
  volume_id   = aws_ebs_volume.mongo_storage.id
}

```

---

### **`variables.tf`**: Variables file

```hcl
variable "aws_region" {
  description = "The AWS region where the resources will be created."
  type        = string
}

variable "vpc_id" {
  description = "The VPC ID where the resources will be created."
  type        = string
}

variable "subnet_ids" {
  description = "The list of Subnet IDs where the EC2 instances will be deployed."
  type        = list(string)
}

variable "allowed_cidr_blocks" {
  description = "The list of allowed CIDR blocks to access the MongoDB instance."
  type        = list(string)
}

variable "db_name" {
  description = "The name for the MongoDB instance."
  type        = string
}

variable "allocated_storage" {
  description = "The size of the EBS volume in GB."
  type        = number
}

variable "instance_class" {
  description = "The EC2 instance type to use for MongoDB."
  type        = string
}

variable "engine_version" {
  description = "MongoDB engine version to be used."
  type        = string
}

variable "key_name" {
  description = "The SSH key pair name to access the EC2 instance."
  type        = string
}
```

---

### **`terraform.tfvars`**: The values for variables

```hcl
aws_region          = "us-west-2"
vpc_id              = "vpc-00249b4c86a64b3eb"
subnet_ids          = ["subnet-04ffa5f09ad7574d6", "subnet-0742c23490882d832"]
allowed_cidr_blocks = ["10.252.237.160/28", "10.252.237.176/28"]
db_name             = "cmlab-mongodb-test"
allocated_storage   = 10
instance_class      = "db.t3.micro"
engine_version      = "17.4"
key_name            = "your-key-pair-name"  # Replace with your SSH key pair name
```

---

### **`outputs.tf`**: Outputs for easy reference

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = var.vpc_id
}

output "subnet_ids" {
  description = "Subnets IDs"
  value       = var.subnet_ids
}

output "mongo_instance_id" {
  description = "MongoDB EC2 Instance ID"
  value       = aws_instance.mongo_instance.id
}

output "mongo_private_ip" {
  description = "Private IP of the MongoDB instance"
  value       = aws_instance.mongo_instance.private_ip
}

output "mongo_storage_id" {
  description = "MongoDB EBS Volume ID"
  value       = aws_ebs_volume.mongo_storage.id
}
```

---

### **Explanation of Changes**:

1. **VPC and Subnet Usage**: We now use the existing `vpc_id` and `subnet_ids` passed in through the `tfvars` file. The EC2 instance will be launched in one of the existing subnets (using the first subnet from `subnet_ids`).

2. **Security Group**: The security group is configured to allow MongoDB access only from the specified CIDR blocks (`allowed_cidr_blocks`), ensuring only specific IP ranges can connect to the MongoDB instance.

3. **MongoDB EC2 Instance**: The EC2 instance is set up to use the `db.t3.micro` instance type and launch MongoDB using the user data script. The instance is configured to **not** have a public IP (`associate_public_ip_address = false`), ensuring it is accessible only via the private network.

4. **EBS Volume**: An EBS volume is attached to the MongoDB instance with a size of `allocated_storage` (10 GB as per your input).

---

### **4. Apply Terraform Configuration**

1. Initialize Terraform:

   ```bash
   terraform init
   ```

2. Review the changes:

   ```bash
   terraform plan
   ```

3. Apply the configuration to provision the resources:

   ```bash
   terraform apply
   ```

   Confirm the action by typing `yes` when prompted.

---

### **5. Access MongoDB Instance**

Since the MongoDB instance is within a private subnet and has no public IP, you can connect to it via a **bastion host** (an EC2 instance with a public IP within the same VPC), or use **VPC peering** or **VPN** for direct access.

---

Let me know if you need further modifications or more details about specific steps!
