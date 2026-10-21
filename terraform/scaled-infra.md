# Creating 50 EC2 Instances with Terraform Without Duplicating Code

Creating 50 EC2 instances does not require 50 Terraform resource blocks. One block can describe the common configuration, while `count` or `for_each` determines how many instances Terraform manages.

The more important decision is how Terraform identifies those servers. That choice affects what happens when you add, remove, or change one later.

## Choose between numbered instances and named servers

For similar instances managed as a numbered collection, `count = 50` is a reasonable starting point. Terraform gives them addresses such as:

```text
aws_instance.app[0]
aws_instance.app[1]
...
aws_instance.app[49]
```

Increasing the count to 51 adds another instance. Reducing it to 49 removes the highest-indexed instance.

Problems arise when those indexes also refer to positions in a changing list. Removing an entry from the middle shifts later values, potentially causing Terraform to change or replace resources you intended to keep.

Use `for_each` when individual servers need stable identities. It identifies instances using map keys, producing addresses such as `aws_instance.server["orders-api"]`. Removing another key does not shift that address. Renaming the key, however, changes the identity unless you explicitly migrate the address. [HashiCorp Help Center](https://support.hashicorp.com/hc/en-us/articles/31348158569363-Terraform-count-versus-for-each-meta-argument?utm_source=chatgpt.com)

The distinction is about identity, not simply whether the servers have different configurations.

## Define common settings once

Suppose you manage application servers with different instance sizes and subnet placements.

The following simplified example assumes an initialized Terraform project with the HashiCorp AWS provider configured, AWS credentials available, and existing networking. The AMI must match the Region and instance architecture; security groups must belong to the subnet’s VPC.

```hcl
variable "servers" {
  type = map(object({
    instance_type = string
    subnet_id     = string
  }))
}

resource "aws_instance" "server" {
  for_each = var.servers

  ami                    = var.ami_id
  instance_type          = each.value.instance_type
  subnet_id              = each.value.subnet_id
  vpc_security_group_ids = var.security_group_ids

  root_block_device {
    encrypted = true
  }

  tags = {
    Name        = each.key
    Environment = var.environment
  }
}
```

Declare the shared inputs `ami_id`, `security_group_ids`, and `environment` separately. Then provide server-specific values in a `.tfvars` file.

This illustrative input contains two entries and placeholder subnet IDs:

```hcl
servers = {
  orders-api = {
    instance_type = "t3.small"
    subnet_id     = "subnet-REPLACE-A"
  }
  invoice-worker = {
    instance_type = "t3.medium"
    subnet_id     = "subnet-REPLACE-B"
  }
}
```

Fifty map entries produce 50 managed instances without changing the resource block. Keep keys tied to a server’s logical identity rather than a setting, such as its instance size, that may change.

A reusable module becomes useful when this configuration also includes IAM instance profiles, additional volumes, bootstrap configuration, and standard tags. Modules package that configuration; `count` and `for_each` provide repetition.

## Protect state and separate environments

Terraform state records the relationship between resource addresses and actual AWS resources. A team needs shared state and locking to prevent competing operations from updating it concurrently.

For a new S3 backend configuration, use S3 lockfile support. DynamoDB-based locking is now deprecated. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/backend/s3?utm_source=chatgpt.com)

With Terraform 1.10 or later, a backend configuration can include:

```hcl
terraform {
  backend "s3" {
    bucket       = "REPLACE-WITH-STATE-BUCKET"
    key          = "prod/compute/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

Create the bucket beforehand, enable versioning, and grant the required state and lockfile permissions. Restrict access because state can contain sensitive values.

Give development and production separate state and appropriate access boundaries. Terraform CLI workspaces separate state, but they are not sufficient isolation when environments require separate credentials and access controls. [HashiCorp Developer](https://docs.hashicorp.com/terraform/cli/workspaces?utm_source=chatgpt.com)

## Review changes before creating the fleet

Initialize the project, validate it, and review a saved execution plan:

```bash
terraform init
terraform fmt
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

Check instance counts, replacements, subnet placement, security rules, and cost before applying. Confirm that EC2 quotas, subnet IP capacity, and instance availability can support the deployment. Protect saved plans because they can contain sensitive information.

Finally, distinguish individually managed servers from interchangeable application capacity. If the requirement is “keep 50 healthy workers running,” an Auto Scaling group may be a better fit. Terraform can define the group and launch template while AWS manages instance replacement.