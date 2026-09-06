# Troubleshooting log

Five failures I hit deploying this stack, in the order I hit them. Each one has
the error as it appeared, what was actually wrong, and what fixed it. They are
here because the errors are not the interesting part — the gap between what the
message says and what is actually broken is.

## 1. Key pair not found

```
Error: creating EC2 Instance: operation error EC2: RunInstances,
api error InvalidKeyPair.NotFound: The key pair 'Macbook_Air_Key' does not exist
```

**Cause.** Key pairs are per-region. I had created the pair in `us-east-1` from
the console, but the provider had no `region` set, so Terraform fell back to the
CLI's default region and looked for the key there.

**Fix.** Pin the region in the provider block instead of inheriting it from
whatever the environment happens to be:

```hcl
provider "aws" {
  region = "us-east-1"
}
```

Inheriting the region is the underlying mistake, and it comes back in section 3
in a more expensive form.

## 2. S3 endpoint type mismatch

```
api error InvalidParameter: Endpoint type (Gateway) does not match
available service types ([Interface])
```

**Cause.** `vpc_endpoint_type` was not declared, so the endpoint defaulted to
`Interface`. S3 supports both types and they are not interchangeable: the
interface endpoint is an ENI with an hourly charge, the gateway endpoint is a
route-table entry with no charge. The whole point of this lab is the second one.

**Fix.** Declare the type explicitly.

```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.us-east-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]
}
```

## 3. The ghost VPC — state pointing at another region

```
api error InvalidVpcID.NotFound: The vpc ID 'vpc-074a0a66bbc190464' does not exist
```

**Cause.** The consequence of section 1. My first `terraform apply` ran without
a declared region and built the VPC in `us-west-2`. When I then pinned the
provider to `us-east-1`, the local `terraform.tfstate` still held the IDs of the
`us-west-2` resources, so Terraform tried to create subnets inside a VPC that
does not exist in the new region.

**Fix, and the part that matters.** The tempting move is to delete the state
file, because the error goes away. It goes away because Terraform has forgotten
about the resources — not because they are gone. They stay running in
`us-west-2` with nothing tracking them, and a NAT Gateway bills by the hour
whether or not anyone remembers it exists.

So the state file is the thing to protect, not the thing to delete:

1. Point the provider back at the wrong region, `us-west-2`.
2. `terraform destroy` — the state still matches reality there, so this actually
   removes the resources.
3. Point the provider at `us-east-1`.
4. `terraform apply`.

## 4. Bastion deployed but unreachable

**Symptom.** The bastion came up and the apply succeeded, but
`bastion_public_ip` came back empty, so there was nothing to SSH to.

**Cause.** `map_public_ip_on_launch = true` on the subnet is a default for
instances launched in it, and it is not the only thing that decides the outcome.

**Fix.** Set it on the instance, where it is not a default but an instruction:

```hcl
resource "aws_instance" "bastion" {
  # ...
  associate_public_ip_address = true
}
```

## 5. Agent forwarding refused on the jump

```
ec2-user@<bastion-ip>: Permission denied (publickey,gssapi-keyex,gssapi-with-mic)
```

**Cause.** I was connecting with `ssh -A` to forward the agent through the
bastion to the private host, but the key was not loaded into the local agent —
having the file on disk is not the same as the agent holding it.

**Fix, on macOS.** Load the key into the keychain and the agent before
connecting:

```bash
ssh-add --apple-use-keychain ~/.ssh/your-key.pem
ssh -A ec2-user@<bastion-public-ip>
```

Copying the private key onto the bastion also works and is the wrong answer: it
puts the key that opens the private tier on the one host that is exposed to the
internet.
