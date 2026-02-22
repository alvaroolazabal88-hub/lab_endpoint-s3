# AWS Multi-AZ VPC: Private Connectivity & Optimized S3 Access 🏗️

## Technical Summary 📋
This project leverages **Terraform** to deploy a highly available and secure network infrastructure on AWS. The architecture implements strict layer segregation using Public and Private subnets distributed across multiple **Availability Zones (AZ)**.

The design focuses on operational and cost efficiency by integrating an **S3 Gateway Endpoint**. This allows private and free communication between internal instances and Amazon S3, bypassing the NAT Gateway and reducing data transfer costs.



## Network Architecture 🗺️

| Component | Technical Function |
| :--- | :--- |
| **VPC** 🏢 | Isolated logical network with a `10.0.0.0/16` address space. |
| **Public Subnets** 🌐 | 2 subnets (Multi-AZ) with direct Internet access via **Internet Gateway (IGW)**. |
| **Private Subnets** 🔒 | 2 subnets (Multi-AZ) isolated for sensitive resources (Databases, App Servers). |
| **NAT Gateway** 🌉 | Provides secure outbound Internet access for private subnets (One-way). |

### The "Master Stroke": S3 Gateway Endpoint 📦
To optimize the AWS bill, we have implemented a **Gateway Endpoint**. This injects a specific prefix list into the private route tables to direct S3 traffic through the AWS internal network.

* **Cost Savings**: $0 data transfer charges for S3 traffic (avoids NAT Gateway processing fees). 💸
* **Performance**: Lower latency by staying off the public internet. ⚡
* **Security**: Traffic never leaves the Amazon network backbone. 🛡️



---

## Deployment Guide 🛠️

1. **Initialize Terraform**:
   Prepare the working directory and download providers.
   ```bash
   terraform init
   terraform plan 
   terraform apply