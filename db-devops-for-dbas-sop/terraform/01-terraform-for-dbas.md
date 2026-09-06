# Terraform for PostgreSQL DBAs

## 1. What is Terraform?

Terraform is an Infrastructure as Code (IaC) tool. A DBA uses Terraform when the database environment itself is part of infrastructure that must be created, changed, reviewed and reproduced from code.

Terraform works through providers. The AWS provider translates Terraform configuration into AWS API operations. Terraform also maintains state so it can map configuration resources to real infrastructure. The normal workflow is `init -> plan -> apply`. citeturn0search8turn0search13turn0search0

## 2. DBA mental model

Think of Terraform as:

> **"How do I build and maintain the database platform?"**

Not:

> "How do I run SQL inside PostgreSQL?"

Examples:

- create an RDS instance
- create an Aurora cluster
- create a DB subnet group
- create security groups
- create parameter groups
- configure monitoring-related infrastructure
- create IAM policies/roles used by database services
- create Secrets Manager infrastructure
- create networking around the database

Terraform normally does not replace SQL, PostgreSQL administration or Liquibase.

---

# 3. Terraform nomenclature

## Provider

A plugin that allows Terraform to communicate with a platform.

```hcl
provider "aws" {
  region = "us-east-1"
}
```

Production DBA use:

- AWS provider for RDS/Aurora
- potentially other providers for DNS, monitoring, secrets or cloud services

## Resource

A managed infrastructure object.

```hcl
resource "aws_db_instance" "postgres" {
  ...
}
```

Think:

> "Terraform owns this infrastructure object."

## Data source

Reads information about an existing object.

```hcl
data "aws_vpc" "existing" {
  ...
}
```

Think:

> "Terraform needs information about this object, but this block is not creating it."

## Variable

An input to make configuration reusable.

```hcl
variable "db_instance_class" {
  type = string
}
```

## Local

A calculated/reusable value inside configuration.

```hcl
locals {
  environment = "prod"
}
```

## Output

A value exposed after Terraform execution.

```hcl
output "db_endpoint" {
  value = aws_db_instance.postgres.address
}
```

## Module

A reusable collection of Terraform configuration. Modules allow infrastructure patterns to be standardized across environments. citeturn0search6

Typical production model:

```text
modules/
  rds-postgresql/
  aurora-postgresql/
  networking/
```

## State

Terraform's record of the relationship between Terraform resource addresses and real infrastructure. State is critical to planning changes. Remote state with locking is normally preferred for team environments. citeturn0search0

## Backend

Where Terraform state is stored.

Production examples:

- HCP Terraform
- Terraform Enterprise
- S3-based remote state with appropriate locking/coordination

## Provider version

Controls which provider implementation Terraform uses.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "6.63.0"
    }
  }
}
```

## `.terraform.lock.hcl`

Records selected provider versions/checksums. Commit it to Git.

## Plan

A preview of infrastructure changes.

```bash
terraform plan
```

## Apply

Executes the planned infrastructure changes.

```bash
terraform apply
```

## Import

Brings an existing infrastructure object under Terraform management/state. Modern Terraform also supports configuration-driven import with `import` blocks and a plan/apply workflow. citeturn0search1turn0search5

---

# 4. Terraform lifecycle from a DBA perspective

```text
Requirement
    |
    v
Terraform code
    |
    v
terraform fmt
    |
    v
terraform validate
    |
    v
terraform init
    |
    v
terraform plan
    |
    v
Code review / approval
    |
    v
terraform apply
    |
    v
AWS infrastructure
    |
    v
PostgreSQL service
```

Never treat `terraform apply` as equivalent to executing a SQL deployment.

---

# 5. Important commands

## Format

```bash
terraform fmt
terraform fmt -check
```

## Initialize

```bash
terraform init
```

Initializes the working directory, backend, providers and modules. citeturn0search13

## Validate

```bash
terraform validate
```

Checks configuration syntax and internal consistency.

## Plan

```bash
terraform plan
```

Production DBA use:

> Review before changing an RDS instance, subnet group, security group or parameter group.

## Save a plan

```bash
terraform plan -out=tfplan
```

Then:

```bash
terraform apply tfplan
```

Useful in controlled CI/CD.

## Apply

```bash
terraform apply
```

## Destroy

```bash
terraform destroy
```

Extremely sensitive in production. Never use casually against production.

## Show state

```bash
terraform show
terraform show -json
```

## List resources in state

```bash
terraform state list
```

## Inspect a resource

```bash
terraform state show aws_db_instance.postgres
```

## Output

```bash
terraform output
terraform output -json
```

## Providers

```bash
terraform providers
```

Shows provider requirements discovered from configuration. citeturn0search12

## Refresh/reconcile awareness

Modern Terraform workflows refresh infrastructure information as part of operations. Do not manually edit `terraform.tfstate`; use Terraform state commands when state manipulation is necessary. citeturn0search0

## Import

Legacy CLI style:

```bash
terraform import aws_db_instance.postgres <RDS_IDENTIFIER>
```

Modern configuration-driven style:

```hcl
import {
  to = aws_db_instance.postgres
  id = "<RDS_IDENTIFIER>"
}
```

Then:

```bash
terraform plan
terraform apply
```

Import does not magically create the correct long-term configuration. The resource configuration still needs to represent the desired lifecycle. citeturn0search5

---

# 6. Terraform commands DBA cheat sheet

| Command | DBA purpose |
|---|---|
| `terraform fmt` | Standardize code |
| `terraform validate` | Catch configuration errors |
| `terraform init` | Initialize providers/backend |
| `terraform plan` | Review infrastructure impact |
| `terraform apply` | Execute infrastructure change |
| `terraform destroy` | Remove infrastructure |
| `terraform state list` | See managed resources |
| `terraform state show` | Inspect one resource |
| `terraform show` | Inspect state/plan |
| `terraform output` | Retrieve outputs |
| `terraform providers` | Inspect providers |
| `terraform import` | Adopt existing infrastructure |

---

# 7. Where Terraform is used in real production

## Incident: RDS storage pressure

Symptom:

```text
Production PostgreSQL storage approaching capacity.
```

DBA actions:

1. Confirm whether storage expansion is appropriate.
2. Review the Terraform configuration.
3. Change the infrastructure parameter in code.
4. Run:

```bash
terraform plan
```

5. Review the proposed RDS modification.
6. Obtain approval.
7. Apply through the controlled pipeline.

Terraform is used because this is an infrastructure lifecycle change.

---

## Incident: Wrong security group

Symptom:

```text
Application cannot connect to PostgreSQL.
```

DBA discovers that the application security group is missing from the RDS security group's allowed source.

Terraform is appropriate when the security group is Terraform-managed.

Flow:

```text
Incident
  |
  v
Identify SG rule
  |
  v
Fix Terraform code
  |
  v
plan
  |
  v
review
  |
  v
apply
```

Do not make an undocumented console change and leave Git inconsistent.

---

## Incident: Production DB created manually

Symptom:

```text
RDS exists in AWS but Terraform does not manage it.
```

Use import.

Terraform's import workflow exists specifically for bringing existing infrastructure under management. citeturn0search1turn0search10

---

## Incident: Parameter group change

Example:

```text
A PostgreSQL parameter must be changed according to an approved performance/configuration change.
```

If the parameter group is managed by Terraform:

```text
Terraform -> parameter group -> RDS
```

If the change is a SQL-level database setting rather than infrastructure configuration, use PostgreSQL administration/Ansible or SQL instead.

---

# 8. What Terraform should NOT do

Do not use Terraform as the normal mechanism for:

- creating application tables
- adding indexes as application releases
- running arbitrary SQL migration scripts
- killing PostgreSQL sessions
- investigating active sessions
- performing routine VACUUM/ANALYZE workflows
- replacing Liquibase
- replacing PostgreSQL monitoring

Those belong to SQL/PostgreSQL tooling, Ansible, Liquibase or monitoring systems.

---

# 9. Production DBA checklist

Before `apply`:

```text
[ ] Is this infrastructure or database schema?
[ ] Is the resource Terraform-managed?
[ ] Is the state healthy?
[ ] Is the plan expected?
[ ] Does the plan show replacement?
[ ] Does it show unexpected destroy?
[ ] Is the environment correct?
[ ] Has the plan been reviewed?
[ ] Is rollback/recovery understood?
```

If Terraform proposes:

```text
-/+ destroy and create replacement
```

stop and investigate before applying.

---

# 10. Golden rule

```text
Terraform = infrastructure lifecycle
PostgreSQL/SQL = database operations
Liquibase = database schema lifecycle
Ansible = repeatable DBA operations
Jenkins = orchestration
```
