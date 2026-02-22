Markdown
# 🛠️ AWS Enterprise Architecture Lab: Troubleshooting Guide

This document records the common errors encountered during the deployment of a Multi-AZ VPC architecture with a Bastion Host, NAT Gateway, and VPC Endpoints using Terraform. It serves as a reference for resolving state mismatches, IAM/SSH issues, and provider misconfigurations.

---

## 1. Key Pair Not Found Error
**Error Message:**
> `Error: creating EC2 Instance: operation error EC2: RunInstances, api error InvalidKeyPair.NotFound: The key pair 'Macbook_Air_Key' does not exist`

**Root Cause:**
AWS Key Pairs are region-specific. The AWS CLI or Terraform provider was defaulting to a different region (e.g., `us-west-2`) than where the Key Pair was physically created in the AWS Management Console (e.g., `us-east-1`).

**Solution:**
Explicitly define the AWS Provider and the target region at the top of the `main.tf` file to ensure Terraform looks for resources in the correct location.
```hcl
provider "aws" {
  region = "us-east-1"
}
2. S3 VPC Endpoint Type Mismatch
Error Message:

api error InvalidParameter: Endpoint type (Gateway) does not match available service types ([Interface])

Root Cause:
When defining a VPC Endpoint for S3 without explicitly specifying the vpc_endpoint_type, Terraform or the AWS API might attempt to create an "Interface" endpoint instead of a "Gateway" endpoint, resulting in a type mismatch.

Solution:
Explicitly declare the endpoint type as Gateway in the resource block.

Terraform
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.us-east-1.s3"
  vpc_endpoint_type = "Gateway" # <--- Added this line
  route_table_ids   = [aws_route_table.private.id]
}
3. The "Ghost VPC" (Terraform State Mismatch)
Error Message:

api error InvalidVpcID.NotFound: The vpc ID 'vpc-074a0a66bbc190464' does not exist

Root Cause:
An initial terraform apply was executed without a defined region, causing resources to be deployed in a default region (e.g., us-west-2). When the provider region was updated to us-east-1 in the code, the local terraform.tfstate file still contained the IDs of the resources located in the old region. Terraform tried to deploy subnets into a VPC ID that did not exist in the new region.

Solution:
Do not manually delete the state file, as this leaves "orphan" resources in AWS that generate costs (especially the NAT Gateway).

Revert the provider region in main.tf back to the incorrect region (us-west-2).

Run terraform destroy -auto-approve to safely remove the mistakenly deployed infrastructure.

Change the provider region back to the correct one (us-east-1).

Run terraform apply -auto-approve.

4. Bastion Host Unreachable (No Public IP)
Issue:
The Bastion Host is successfully deployed, but no Public IP is printed in the outputs, making SSH access impossible.

Root Cause:
While the public subnet might have map_public_ip_on_launch = true, it is a best practice (and sometimes required depending on route configurations) to force the EC2 instance to associate a public IP upon creation.

Solution:
Add the associate_public_ip_address argument to the Bastion Host EC2 resource.

Terraform
resource "aws_instance" "bastion" {
  # ... other config
  associate_public_ip_address = true
}
5. SSH Agent Forwarding: Permission Denied
Error Message:

ec2-user@35.170.53.119: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).

Root Cause:
Attempting to connect to the Bastion Host (or jump to the private instance) using the -A (Agent Forwarding) flag, but the local machine's SSH agent does not have the private key loaded into its active memory.

Solution (macOS):
Load the private key into the Apple Keychain and the SSH agent before initiating the SSH connection.