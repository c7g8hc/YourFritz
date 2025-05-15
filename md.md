If you want to set up a **MongoDB Atlas** instance (a fully managed MongoDB service) using **Terraform**, you can leverage the official MongoDB Atlas provider. MongoDB Atlas is a cloud service provided by MongoDB, and it simplifies managing and scaling MongoDB databases.

To provision MongoDB Atlas resources using Terraform, you'll need to follow a few steps. Below is how to adapt your current setup to use MongoDB Atlas for creating a **MongoDB Cluster**.

---

### Steps:

1. **Set up MongoDB Atlas provider**: This requires you to authenticate to MongoDB Atlas using either an API key or MongoDB Atlas credentials.
2. **Create a MongoDB Cluster**: This is your fully-managed MongoDB instance.
3. **Set up Database User**: You may want to create a user to access the database.

---

### **1. Install the MongoDB Atlas Terraform Provider**

First, you need to include the MongoDB Atlas provider in your `main.tf` file.

Here’s an example of how to set up MongoDB Atlas using Terraform.

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

### **2. `main.tf`**: Main Terraform configuration file

```hcl
provider "mongodbatlas" {
  public_key  = var.atlas_public_key
  private_key = var.atlas_private_key
}

# MongoDB Atlas Cluster
resource "mongodbatlas_cluster" "mongo_cluster" {
  project_id       = var.atlas_project_id
  name             = var.db_name
  provider_name    = "AWS"  # Can be AWS, GCP, or Azure
  region_name      = "US_WEST_2"  # Choose your preferred region
  cluster_type     = "REPLICASET"
  disk_size_gb     = 10  # Disk size in GB
  provider_instance_size_name = "M10"  # Instance size (M10 is a good starting point)

  backup_enabled = true

  # Set MongoDB version
  mongo_db_version = "4.2"

  replication_factor = 3  # To create a replica set
  mongo_uri          = "mongodb+srv://<your-cluster>.mongodb.net"  # The URI to connect to your Atlas cluster

  # Encryption at rest
  encryption_at_rest_enabled = true

  tags = {
    Name = var.db_name
  }
}

# MongoDB Atlas Database User
resource "mongodbatlas_database_user" "mongo_user" {
  username   = var.db_username
  password   = var.db_password
  project_id = var.atlas_project_id
  database_names = ["admin", "test"]  # List of databases the user can access
  roles {
    role_name     = "readWrite"
    database_name = "test"
  }
}

# MongoDB Atlas IP Whitelist (optional)
resource "mongodbatlas_project_ip_whitelist_entry" "mongo_ip_whitelist" {
  project_id = var.atlas_project_id
  cidr_block = var.allowed_cidr_blocks
}

output "mongo_cluster_name" {
  description = "MongoDB Atlas Cluster Name"
  value       = mongodbatlas_cluster.mongo_cluster.name
}

output "mongo_cluster_uri" {
  description = "MongoDB Atlas Cluster URI"
  value       = mongodbatlas_cluster.mongo_cluster.mongo_uri
}

output "mongo_database_user" {
  description = "MongoDB Atlas Database User"
  value       = mongodbatlas_database_user.mongo_user.username
}

output "mongo_password" {
  description = "MongoDB Atlas Database User Password"
  value       = var.db_password
  sensitive   = true
}
```

---

### **3. `variables.tf`**: Variable definitions file

```hcl
variable "atlas_public_key" {
  description = "MongoDB Atlas Public API Key"
  type        = string
}

variable "atlas_private_key" {
  description = "MongoDB Atlas Private API Key"
  type        = string
  sensitive   = true
}

variable "atlas_project_id" {
  description = "MongoDB Atlas Project ID"
  type        = string
}

variable "db_name" {
  description = "The name for the MongoDB Atlas cluster."
  type        = string
}

variable "db_username" {
  description = "Master username for the MongoDB Atlas instance."
  type        = string
}

variable "db_password" {
  description = "Master password for the MongoDB Atlas instance."
  type        = string
  sensitive   = true
}

variable "allowed_cidr_blocks" {
  description = "The list of allowed CIDR blocks to access the MongoDB instance."
  type        = list(string)
}
```

---

### **4. `terraform.tfvars`**: The values for your variables

```hcl
atlas_public_key   = "your-atlas-public-key"
atlas_private_key  = "your-atlas-private-key"
atlas_project_id   = "your-atlas-project-id"  # MongoDB Atlas Project ID
db_name            = "cmlab-mongodb-test"
db_username        = "admin"
db_password        = "yourpassword"
allowed_cidr_blocks = ["10.252.237.160/28", "10.252.237.176/28"]
```

> **Important**: You will need to replace `your-atlas-public-key`, `your-atlas-private-key`, and `your-atlas-project-id` with actual values from your MongoDB Atlas account. To get these, you can refer to the [MongoDB Atlas API documentation](https://www.mongodb.com/docs/atlas/reference/api/).

---

### **5. `outputs.tf`**: Outputs for easy access to important resources

```hcl
output "mongo_cluster_name" {
  description = "MongoDB Atlas Cluster Name"
  value       = mongodbatlas_cluster.mongo_cluster.name
}

output "mongo_cluster_uri" {
  description = "MongoDB Atlas Cluster URI"
  value       = mongodbatlas_cluster.mongo_cluster.mongo_uri
}

output "mongo_database_user" {
  description = "MongoDB Atlas Database User"
  value       = mongodbatlas_database_user.mongo_user.username
}

output "mongo_password" {
  description = "MongoDB Atlas Database User Password"
  value       = var.db_password
  sensitive   = true
}
```

---

### **6. Apply Terraform Configuration**

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

### **7. Connecting to MongoDB Atlas**

Once the resources are provisioned, you can get the **MongoDB URI** (`mongo_cluster_uri`) and use it to connect to your MongoDB Atlas cluster. Here’s an example using **Mongo Shell**:

```bash
mongo "mongodb+srv://<your-cluster>.mongodb.net" --username admin --password yourpassword --authenticationDatabase admin
```

Replace `<your-cluster>` with your actual cluster name, and `yourpassword` with the password you specified in `terraform.tfvars`.

---

### Summary of Key Features:

* **MongoDB Atlas Cluster**: A fully managed MongoDB instance.
* **Database User**: Created with specified roles and access to databases.
* **IP Whitelist**: Restricts access to only the specified CIDR blocks.
* **Outputs**: You get essential information like the cluster URI and database user credentials.

---

### Cleanup

To destroy the resources created by Terraform, you can run:

```bash
terraform destroy
```

This will remove the MongoDB Atlas resources and any other infrastructure you have provisioned.

---

Let me know if you need more help or adjustments!
