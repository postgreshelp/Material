# 02 · Concepts + Lab — Terraform: VPC & Aurora

**PostgresHelp Labs**

Every resource block used to stand up the `bt01` network and Aurora PostgreSQL cluster — mental model first, then the lab, then what to verify. Resource identifiers use underscores (`bt01_vpc`); AWS-facing `Name` tags keep hyphens since those are console labels, not code.

> **BEFORE YOU START**
> - ✓ Terraform installed (topic 08) — `terraform -version` works
> - ✓ AWS CLI authenticated (topic 07) — `aws sts get-caller-identity` returns an account
> - ✓ An empty working folder for this project
> - ✓ You know your target AWS region (`us-east-1` in this manual)

---

## 1.1 · Read the existing default VPC

A data source — reads information, creates nothing.

> **MENTAL MODEL**
> Before creating anything, it's worth knowing Terraform can also just *look* at what's already there. A `data` block is a read-only question to AWS — "what does the default VPC look like?" — with zero risk of changing anything.

1. **`data`** — Declares a data source — a read-only lookup, not a resource to manage.
2. **`"aws_vpc"`** — The data source type, provided by the AWS provider — fetches details about a VPC that already exists.
3. **`"default"`** — The local name for this data source, e.g. `data.aws_vpc.default`.
4. **`default = true`** — The filter — "find the VPC flagged as the account's default," not a VPC by name or ID.

**providers.tf**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "6.63.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

**data.tf**
```hcl
data "aws_vpc" "default" {
  default = true
}
```

**outputs.tf**
```hcl
output "default_vpc_id" {
  description = "ID of the existing default VPC"
  value       = data.aws_vpc.default.id
}
```

**Resource address anatomy**
```
data.aws_vpc.default.id
     │        │       │    │
     │        │       │    └── attribute (the VPC ID)
     │        │       └── data source name (your local alias)
     │        └── data source type
     └── keyword: this is a data lookup, not a managed resource
```

> **WHY THIS COMMAND?**
> `terraform apply` here does no provisioning at all — it evaluates the data source and prints the output. Running it is how you confirm the AWS provider is actually authenticated and reachable before you trust it with anything that creates infrastructure.

**Run**
```powershell
PS C:\Users\hp> terraform init
PS C:\Users\hp> terraform apply
```

> **EXPECTED RESULT**
> - **Look for:** "Apply complete! Resources: 0 added, 0 changed, 0 destroyed" followed by `default_vpc_id = "vpc-..."`
> - **0 added?** Correct — a data source reads, it never adds a resource. If you see 1 added here, you've written a `resource` block by mistake.
> - **If it fails:** An auth/credential error here means topic 07 (AWS Authentication) isn't actually working — fix that before continuing, not this file.

```powershell
PS C:\Users\hp> terraform apply
data.aws_vpc.default: Reading...
data.aws_vpc.default: Read complete after 2s [id=vpc-063618af923af9f03]

Changes to Outputs:
  + default_vpc_id = "vpc-063618af923af9f03"

Enter a value: yes

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

default_vpc_id = "vpc-063618af923af9f03"
```

> **⬆ COMMIT & PUSH**
> First commit for this project — includes both the read-default-VPC files above and the import files from 1.5, since they were built in the same initial session before the first push. A `.gitignore` (excluding `.terraform/`, `*.tfstate`) was created first so local state never gets committed.
> ```powershell
> $ git init
> $ git config user.name "Y_USER_NAME"
> $ git config user.email "Y_USER_EMAIL"
> $ git remote add origin https://github.com/postgreshelp/NewTerraform.git
> $ git add .
> $ git status
> $ git commit -m "Add Terraform VPC read and import examples"
> $ git branch -M master
> $ git push -u origin master
> ```

---

## 1.2 · Create a new VPC

A managed resource — Terraform creates and owns this from here on.

> **MENTAL MODEL**
> This is the shift from "asking" to "owning." Once a `resource` block exists in your config and you apply it, Terraform considers that object its responsibility — it will create it if missing, change it if your config changes, and destroy it if you remove the block or run `destroy`.

1. **`resource`** — Declares something Terraform should create, update, and eventually destroy — as opposed to `data`, which only reads.
2. **`"aws_vpc"`** — Resource type — maps directly to an AWS VPC.
3. **`"bt01_vpc"`** — Resource name — the local identifier used elsewhere, e.g. `aws_vpc.bt01_vpc.id`. Underscores, per Terraform convention.
4. **`cidr_block`** — The IP address range, in CIDR notation — `10.0.0.0/16` gives ~65,536 addresses.
5. **`enable_dns_support` / `enable_dns_hostnames`** — Turns on internal DNS — needed for RDS/Aurora endpoints to resolve.
6. **`tags`** — Key/value labels on the AWS resource — hyphens are fine here, this isn't a Terraform identifier.

**aws-create-vpc.tf**
```hcl
resource "aws_vpc" "bt01_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "bt01-vpc"
  }
}
```

**outputs.tf**
```hcl
output "aws_vpc" {
  description = "ID of the newly created VPC"
  value       = aws_vpc.bt01_vpc.id
}
```

**Resource address anatomy**
```
aws_vpc.bt01_vpc
     │       │
     │       └── resource name
     └── resource type

aws_vpc.bt01_vpc.id
     │                 │
     │                 └── resource attribute
     └── resource address
```

> **WHY THIS COMMAND?**
> `terraform plan` is the comparison/preview step: it diffs your config against the state file and shows exactly what would change — before anything real happens. Skipping straight to `apply` means finding out what changes by watching them happen live, which is a bad habit to build.

**Run**
```powershell
PS C:\Users\hp> terraform plan
PS C:\Users\hp> terraform apply
```

> **EXPECTED RESULT**
> - **terraform plan:** "Plan: 1 to add, 0 to change, 0 to destroy." — Terraform found nothing matching this resource yet.
> - **What does it mean:** Terraform compared your desired config against its state file and found the VPC missing — so it proposes creating exactly one thing.
> - **terraform apply:** "Apply complete! Resources: 1 added" and a real `vpc-...` ID printed.

**What changed?**
```
Before                          After
AWS                              AWS
└── (no bt01 VPC)                └── VPC  bt01_vpc
                                       └── (empty — no subnets yet)
```
> Terraform created one VPC and started tracking it in `terraform.tfstate`. It contains no subnets, gateway, or route table yet — that's topic 1.3.

> **⬆ COMMIT & PUSH**
> One commit per milestone from here on — each resource file gets its own push right after its `apply` succeeds, not batched with the next feature.
> ```powershell
> $ git status
> $ git add .
> $ git commit -m "Add a new VPC"
> $ git push -u origin master
> ```

---

## 1.3 · Networking — gateway, subnets, routing

Five resources that give the VPC internet access and split it into public/private zones.

> **MENTAL MODEL**
> A VPC by itself is a sealed box — nothing gets in or out. These five resources are the plumbing: a gateway to the internet, subnets to place things in, and a route table that says which traffic goes where. Miss the association step and a subnet stays sealed even with a gateway attached.

1. **`aws_internet_gateway`** — Attaches an internet gateway to the VPC — required before anything inside can reach the public internet.
2. **`aws_subnet`** — Carves a smaller CIDR range out of the VPC, tied to one AZ. `map_public_ip_on_launch` is what makes a subnet "public."
3. **`aws_route_table` + `route`** — The `0.0.0.0/0 → gateway_id` route sends all outbound traffic through the internet gateway.
4. **`aws_route_table_association`** — Binds a subnet to a route table — without this, the subnet has no internet access regardless of the gateway existing.

**aws-networking.tf**
```hcl
resource "aws_internet_gateway" "bt01_igw" {
  vpc_id = aws_vpc.bt01_vpc.id

  tags = {
    Name = "bt01-igw"
  }
}

resource "aws_subnet" "bt01_public_subnet" {
  vpc_id                  = aws_vpc.bt01_vpc.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "bt01-public-subnet"
  }
}

resource "aws_subnet" "bt01_private_subnet" {
  vpc_id            = aws_vpc.bt01_vpc.id
  availability_zone = "us-east-1b"
  cidr_block        = "10.0.2.0/24"

  tags = {
    Name = "bt01-private-subnet"
  }
}

resource "aws_route_table" "bt01_route_table" {
  vpc_id = aws_vpc.bt01_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.bt01_igw.id
  }

  tags = {
    Name = "bt01-route-table"
  }
}

resource "aws_route_table_association" "bt01_public_subnet_association" {
  subnet_id      = aws_subnet.bt01_public_subnet.id
  route_table_id = aws_route_table.bt01_route_table.id
}

resource "aws_route_table_association" "bt01_private_subnet_association" {
  subnet_id      = aws_subnet.bt01_private_subnet.id
  route_table_id = aws_route_table.bt01_route_table.id
}
```

> **WHY THIS COMMAND?**
> `terraform fmt` before `plan` isn't cosmetic — it's a linter that fixes alignment/indentation automatically, so the diff you review in `plan` is about real changes, not whitespace.

> **EXPECTED RESULT**
> - **terraform plan:** "Plan: 5 to add, 0 to change, 0 to destroy." — one gateway, two subnets, one route table, two associations.
> - **Not 5?** If a resource already exists from a previous partial apply, the count will be lower — check `terraform state list` before applying.

**What changed?**
```
Before                              After
VPC  bt01_vpc                       VPC  bt01_vpc
└── (empty)                         ├── Internet Gateway  bt01_igw
                                     ├── Subnet  bt01_public_subnet   (10.0.1.0/24)
                                     ├── Subnet  bt01_private_subnet  (10.0.2.0/24)
                                     ├── Route Table  bt01_route_table
                                     │     └── 0.0.0.0/0 → bt01_igw
                                     └── both subnets associated to the route table
```
> The VPC now has a real path to the internet, and both subnets are wired to it — nothing is publicly reachable yet, because no compute or database exists in either subnet until 1.4.

```powershell
PS C:\Users\hp> terraform apply
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.
```

> **⬆ COMMIT & PUSH**
> ```powershell
> $ git add .
> $ git commit -m "Add VPC networking: IGW, subnets, route table"
> $ git push
> ```

---

## 1.4 · Aurora PostgreSQL cluster

Subnet group + security group + the cluster and its one instance.

> **MENTAL MODEL**
> Aurora needs to know two things before it can exist: *where* it's allowed to live (the subnet group) and *who's* allowed to talk to it (the security group). The cluster and its instance are separate objects on purpose — a cluster with zero instances is valid (just unusable), which matters more once you add read replicas later.

1. **`aws_db_subnet_group`** — Tells RDS/Aurora which subnets it can use — must span at least two AZs.
2. **`aws_security_group` — `ingress`/`egress`** — `from_port/to_port = 0` with `protocol = "-1"` means **all ports, all protocols** — wide open, fine for a lab, never for production.
3. **`aws_rds_cluster`** — The Aurora cluster — engine, master credentials, database name. `skip_final_snapshot = true` means no backup snapshot on destroy.
4. **`aws_rds_cluster_instance`** — The actual compute node — the cluster alone has no instances until this exists.

**aws-aurora.tf**
```hcl
## subnet group
resource "aws_db_subnet_group" "bt01_aurora" {
  name = "bt01-aurora-subnet-group"

  subnet_ids = [
    aws_subnet.bt01_public_subnet.id,
    aws_subnet.bt01_private_subnet.id
  ]

  tags = {
    Name = "bt01-aurora-subnet-group"
  }
}

## security group
resource "aws_security_group" "bt01_aurora" {
  name        = "bt01-aurora-sg"
  description = "Security group for Aurora PostgreSQL"
  vpc_id      = aws_vpc.bt01_vpc.id

  ingress {
    description = "PostgreSQL"
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

  tags = {
    Name = "bt01-aurora-sg"
  }
}

## aurora cluster
resource "aws_rds_cluster" "bt01_aurora" {
  cluster_identifier = "bt01-aurora"
  engine             = "aurora-postgresql"

  master_username = "postgres"
  master_password = "postgres"

  db_subnet_group_name   = aws_db_subnet_group.bt01_aurora.name
  vpc_security_group_ids = [aws_security_group.bt01_aurora.id]

  database_name = "paylite"

  skip_final_snapshot = true

  tags = {
    Name = "bt01-aurora"
  }
}

resource "aws_rds_cluster_instance" "bt01_aurora" {
  identifier         = "bt01-aurora-instance-1"
  cluster_identifier = aws_rds_cluster.bt01_aurora.id

  instance_class = "db.t3.medium"
  engine         = aws_rds_cluster.bt01_aurora.engine

  tags = {
    Name = "bt01-aurora-instance-1"
  }
}
```

> ⚠️ **Lab-only settings** — hardcoded `master_password = "postgres"` and the `0.0.0.0/0` ingress rule are fine for a throwaway course environment. For anything real: a Secrets Manager-generated password and ingress restricted to the specific CIDR/security group that needs port 5432.

> **EXPECTED RESULT**
> - **Look for:** "Apply complete! Resources: 4 added" — subnet group, security group, cluster, instance.
> - **Timing:** This step takes minutes, not seconds — the instance alone can take 5–8 minutes to reach "available."

**What changed?**
```
Before                              After
VPC  bt01_vpc                       VPC  bt01_vpc
├── Subnets, IGW, route table       ├── Subnets, IGW, route table
└── (no database)                   ├── DB Subnet Group  bt01_aurora
                                     ├── Security Group  bt01_aurora-sg
                                     └── Aurora Cluster  bt01-aurora
                                           └── Instance  bt01-aurora-instance-1  (db.t3.medium)
```
> A running Aurora PostgreSQL instance now exists inside the VPC, reachable on any port from anywhere (per the lab security group) — this is the endpoint topic 09 (Liquibase) and topic 11 (psql) connect to.

```powershell
PS C:\Users\hp> terraform apply
aws_rds_cluster.bt01_aurora: Creation complete after 4m12s
aws_rds_cluster_instance.bt01_aurora: Creation complete after 6m40s

Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
```

> **⬆ COMMIT & PUSH**
> ```powershell
> $ git add .
> $ git commit -m "Add Aurora PostgreSQL cluster"
> $ git push
> ```

---

## 1.5 · Import an existing VPC

Bring a VPC that already exists in AWS under Terraform's management, without recreating it.

> **MENTAL MODEL**
> `terraform import` only writes to the state file — "this AWS object and this resource address are now the same thing." It does *not* read your `.tf` file's arguments and does not check they match the real object. That gap is exactly what the troubleshooting box below is about.

1. **`terraform import`** — A CLI command (not a config block) that maps a real AWS object to a resource address already defined in your `.tf` files.
2. **`aws_vpc.default_import`** — The resource address you're importing *into* — the block must already exist in your config.

**aws-default-import.tf**
```hcl
resource "aws_vpc" "default_import" {
  cidr_block           = "10.10.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "B02-VPC"
  }
}
```

> **WHY THIS COMMAND?**
> `aws ec2 describe-vpcs` runs first because import needs the *real* CIDR to write a config block that matches — guessing it, or copying it from memory, is exactly how the mismatch below happens.

> **EXPECTED RESULT**
> - **Look for:** "Import successful!" — this confirms the state file now links, it does not confirm your config matches.
> - **Next step:** Always run `terraform plan` immediately after an import. If it proposes changes, your `.tf` block doesn't match reality yet.

```powershell
PS C:\Users\hp> aws ec2 describe-vpcs --vpc-ids vpc-02f476ec787e998c6
10.10.0.0/16

PS C:\Users\hp> terraform import aws_vpc.default_import vpc-02f476ec787e998c6
aws_vpc.default_import: Importing from ID "vpc-02f476ec787e998c6"...
aws_vpc.default_import: Import prepared!
aws_vpc.default_import: Refreshing state... [id=vpc-02f476ec787e998c6]

Import successful!
```

> ⚠️ **REAL ISSUE ENCOUNTERED**
> ```
> terraform import aws_vpc.default_import vpc-02f476ec787e998c6
>       │
>       ▼
> State now knows this resource (id + attributes as they exist in AWS)
>       │
>       ▼
> Your .tf config block was written with a CIDR that doesn't match
>       │
>       ▼
> terraform plan
>       │
>       ▼
> Terraform proposes to CHANGE (or even replace) the VPC to match your .tf file
> ```
> **Lesson —** importing a resource into state does not make your Terraform configuration match it. Import only links an ID; the config's arguments are still whatever you typed. Always run `plan` right after an import — if it shows changes, fix the `.tf` file's values to match reality (usually by copying them from `terraform show` or the AWS console), don't apply blind.

> **⬆ COMMIT & PUSH**
> Already pushed — this file was part of the first commit back in 1.1 ("Add Terraform VPC read and import examples"), since both were built before the first push happened.

---

## Can I explain this? (topic 01 — Terraform)

- [ ] What is a Terraform resource, and how is it different from a data source?
- [ ] What is a resource type vs. a resource name vs. a resource address?
- [ ] Why does Terraform need a state file at all?
- [ ] What does `terraform plan` actually do that `apply` doesn't?
- [ ] Why does importing a resource NOT guarantee your config matches it?
- [ ] Why commit after each resource file's apply, instead of batching everything into one push at the end?
- [ ] Without looking back — sketch the VPC → subnets → route table → Aurora dependency chain from memory.

---

All resource identifiers on this page use underscores (`bt01_vpc`) to match Terraform naming convention. AWS-facing `Name` tags keep hyphens intentionally.

*PostgresHelp Labs · Build. Operate. Automate. Grow.*
