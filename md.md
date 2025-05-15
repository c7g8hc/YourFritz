If you want to set up an **AWS Managed MongoDB** instance, you can use **Amazon DocumentDB**, which is fully compatible with MongoDB, but is a fully managed service provided by AWS.

Here’s how you can set up an **Amazon DocumentDB** instance using **Terraform** with an existing VPC, subnets, and other parameters as provided.

---

### **Directory Structure:**

```
/terraform
├── main.tf
├── variables.tf
├── terraform.tfvars
└── outputs.tf
```

---

### **1. `main.tf`**: Main Terraform configuration file

```hcl
provider "aws" {
  region = var.aws_region
}

# Amazon DocumentDB Cluster
resource "aws_docdb_cluster" "mongo_cluster" {
  cluster_identifier      = var.db_name
  engine                 = "docdb"
  engine_version         = var.engine_version
  master_username        = var.db_username
  master_password        = var.db_password
  skip_final_snapshot    = true
  storage_encrypted      = true
  db_cluster_parameter_group_name = "default.docdb3.6"  # Adjust to your MongoDB-compatible version

  vpc_security_group_ids = [aws_security_group.mongo_sg.id]

  tags = {
    Name = var.db_name
  }
}

# Amazon DocumentDB Instance
resource "aws_docdb_cluster_instance" "mongo_instance" {
  cluster_identifier = aws_docdb_cluster.mongo_cluster.id
  instance_class     = var.instance_class
  engine             = "docdb"
  instance_identifier = "${var.db_name}-instance"
  publicly_accessible = false  # Ensure the instance is not publicly accessible

  tags = {
    Name = "${var.db_name}-instance"
  }
}

# Security Group for MongoDB (to restrict access)
resource "aws_security_group" "mongo_sg" {
  name        = "mongo_security_group"
  description = "Security group for Amazon DocumentDB with restricted access"
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

# Outputs
output "mongo_cluster_id" {
  description = "Amazon DocumentDB Cluster ID"
  value       = aws_docdb_cluster.mongo_cluster.id
}

output "mongo_instance_id" {
  description = "Amazon DocumentDB Instance ID"
  value       = aws_docdb_cluster_instance.mongo_instance.id
}

output "mongo_endpoint" {
  description = "MongoDB Cluster Endpoint"
  value       = aws_docdb_cluster.mongo_cluster.endpoint
}

output "mongo_port" {
  description = "MongoDB Port"
  value       = 27017
}
```

---

### **2. `variables.tf`**: Variable definitions file

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
  description = "The list of Subnet IDs where the DocumentDB cluster will be deployed."
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

variable "instance_class" {
  description = "The instance type for DocumentDB instance."
  type        = string
}

variable "engine_version" {
  description = "DocumentDB engine version to be used."
  type        = string
}

variable "db_username" {
  description = "Master username for the MongoDB instance."
  type        = string
}

variable "db_password" {
  description = "Master password for the MongoDB instance."
  type        = string
  sensitive   = true
}
```

---

### **3. `terraform.tfvars`**: Values for your variables

```hcl
aws_region          = "us-west-2"
vpc_id              = "vpc-00249b4c86a64b3eb"
subnet_ids          = ["subnet-04ffa5f09ad7574d6", "subnet-0742c23490882d832"]
allowed_cidr_blocks = ["10.252.237.160/28", "10.252.237.176/28"]
db_name             = "cmlab-mongodb-test"
instance_class      = "db.t3.micro"
engine_version      = "3.6.0"  # Choose an appropriate version
db_username         = "admin"  # Choose your master username
db_password         = "yourpassword"  # Set your master password here
```

---

### **4. `outputs.tf`**: Outputs for easy access to important resources

```hcl
output "mongo_cluster_id" {
  description = "MongoDB Cluster ID"
  value       = aws_docdb_cluster.mongo_cluster.id
}

output "mongo_instance_id" {
  description = "MongoDB Instance ID"
  value       = aws_docdb_cluster_instance.mongo_instance.id
}

output "mongo_endpoint" {
  description = "MongoDB Cluster Endpoint"
  value       = aws_docdb_cluster.mongo_cluster.endpoint
}

output "mongo_port" {
  description = "MongoDB Port"
  value       = 27017
}
```

---

### **Explanation of the Terraform Setup**:

1. **Amazon DocumentDB Cluster** (`aws_docdb_cluster`):

   * This resource creates a fully managed MongoDB-compatible cluster on Amazon DocumentDB.
   * The `engine_version` parameter is set to `"3.6.0"` (DocumentDB is compatible with MongoDB 3.6). Adjust this version if needed.
   * **Master username** and **password** are provided to access the MongoDB database.

2. **Amazon DocumentDB Instance** (`aws_docdb_cluster_instance`):

   * This is an instance running inside the DocumentDB cluster. It is set to **not be publicly accessible** (`publicly_accessible = false`).

3. **Security Group** (`aws_security_group`):

   * The security group only allows inbound access on port `27017` (the default MongoDB port) from the specified CIDR blocks (`allowed_cidr_blocks`).
   * **Outbound traffic** is unrestricted to allow the cluster to communicate with the internet or other services.

4. **Outputs**:

   * The `mongo_cluster_id`, `mongo_instance_id`, `mongo_endpoint`, and `mongo_port` are exposed as output values to help you connect to the DocumentDB instance.

---

### **5. Apply Terraform Configuration**:

1. **Initialize Terraform**:

   ```bash
   terraform init
   ```

2. **Review the Terraform plan**:

   ```bash
   terraform plan
   ```

3. **Apply the Terraform plan**:

   ```bash
   terraform apply
   ```

   Confirm the action by typing `yes` when prompted.

---

### **6. Connecting to Amazon DocumentDB**

Once the Terraform script has successfully applied, you will get the **Cluster Endpoint** and **Port**. You can use this information to connect to the MongoDB-compatible instance using tools like **Mongo Shell**, **MongoDB Compass**, or any MongoDB client.

For example:

```bash
mongo --ssl --host <your-cluster-endpoint> --username admin --password yourpassword --authenticationDatabase admin --port 27017
```

Make sure to replace `<your-cluster-endpoint>` with the actual endpoint and `yourpassword` with the password you configured in `terraform.tfvars`.

---

### **7. Cleanup**

If you no longer need the resources, you can destroy them to avoid unnecessary charges:

```bash
terraform destroy
```

---

### Summary of Key Changes:

* **Amazon DocumentDB** is used instead of self-managing MongoDB.
* **VPC, subnets**, and **CIDR blocks** are passed through `terraform.tfvars` to ensure the resources are created in the existing infrastructure.
* The **DocumentDB cluster** is configured with secure access, using private subnets and security groups to restrict access to specific CIDR blocks.

Let me know if you need further adjustments or more details!
