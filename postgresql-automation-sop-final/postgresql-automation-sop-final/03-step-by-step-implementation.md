# PostgreSQL Automation Project --- Step-by-Step Implementation

This document rewrites the hands-on implementation as one progressive
project.

## Final repository structure

``` text
ansible-postgresql/
│
├── .gitignore
├── ansible.cfg
│
├── provider.tf
├── variables.tf
├── terraform.tfvars
├── data.tf
├── aws-vpc.tf
├── networking.tf
├── aurora.tf
├── outputs.tf
│
├── inventory/
│   ├── hosts
│   └── group_vars/
│       └── postgresql.yml
│
├── playbooks/
│   ├── postgresql_role.yml
│   ├── postgresql_admin.yml
│   ├── postgresql_admin_full.yml
│   └── liquibase_deploy.yml
│
├── liquibase/
│   ├── liquibase.properties
│   ├── db.changelog-master.yaml
│   └── changelog/
│       ├── 001-create-schema.sql
│       └── 002-create-table.sql
│
├── Jenkinsfile
└── monitoring2.sh
```

> The lab uses simple credentials for learning. Do not commit real
> production passwords to Git.

------------------------------------------------------------------------

# PART 1 --- Start the Git Repository

## Step 1 --- Create the project

On the local Windows machine:

``` powershell
cd C:\Users\hp\Documents\Course
mkdir ansible-postgresql
cd ansible-postgresql
code .
```

If practicing Git from scratch, remove only the local Git metadata:

``` powershell
Remove-Item -Recurse -Force .git
```

If you also want a fresh Terraform provider/cache directory:

``` powershell
Remove-Item -Recurse -Force .terraform
```

Do **not** remove `terraform.tfstate` unless you intentionally want to
forget Terraform's resource tracking.

------------------------------------------------------------------------

# PART 2 --- Terraform

## Step 2 --- Provider

Create `provider.tf`:

``` hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

Run:

``` powershell
terraform init
terraform validate
```

------------------------------------------------------------------------

## Step 3 --- Read the default VPC

Create `data.tf`:

``` hcl
data "aws_vpc" "default" {
  default = true
}
```

Create/update `outputs.tf`:

``` hcl
output "default_vpc_id" {
  description = "ID of the existing default VPC"
  value       = data.aws_vpc.default.id
}
```

Run:

``` powershell
terraform fmt
terraform validate
terraform plan
terraform apply
terraform output
```

Concept:

``` text
data = read existing infrastructure
```

------------------------------------------------------------------------

## Step 4 --- Understand import

If an existing VPC must be brought under Terraform management:

``` hcl
resource "aws_vpc" "b02_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "B02-VPC"
  }
}
```

Import:

``` powershell
terraform import aws_vpc.b02_vpc vpc-xxxxxxxx
```

Inspect:

``` powershell
terraform state list
terraform show
terraform plan
```

Remember:

``` text
data
  = read

resource
  = manage

terraform import
  = associate existing resource with Terraform state
```

Import does not automatically create the desired Terraform configuration
for you.

------------------------------------------------------------------------

# PART 3 --- Build the Lab VPC

## Step 5 --- Create the VPC

Create `aws-vpc.tf`:

``` hcl
resource "aws_vpc" "bt01_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "bt01-vpc"
  }
}
```

Run:

``` powershell
terraform fmt
terraform validate
terraform plan
```

Review the plan.

Then:

``` powershell
terraform apply
```

------------------------------------------------------------------------

# PART 4 --- Internet Gateway

## Step 6 --- Create the Internet Gateway

Add to `networking.tf`:

``` hcl
resource "aws_internet_gateway" "bt01_igw" {
  vpc_id = aws_vpc.bt01_vpc.id

  tags = {
    Name = "bt01-igw"
  }
}
```

Run:

``` powershell
terraform fmt
terraform validate
terraform plan
terraform apply
```

------------------------------------------------------------------------

# PART 5 --- Subnets

## Step 7 --- Create two subnets

Add:

``` hcl
resource "aws_subnet" "bt01_public_subnet" {
  vpc_id            = aws_vpc.bt01_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  map_public_ip_on_launch = true

  tags = {
    Name = "bt01-public-subnet"
  }
}

resource "aws_subnet" "bt01_private_subnet" {
  vpc_id            = aws_vpc.bt01_vpc.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"

  map_public_ip_on_launch = false

  tags = {
    Name = "bt01-private-subnet"
  }
}
```

The two Availability Zones are important because an RDS/Aurora DB subnet
group must cover sufficient AZs.

------------------------------------------------------------------------

# PART 6 --- Route Table

## Step 8 --- Create route table

Add:

``` hcl
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
```

Associate both subnets:

``` hcl
resource "aws_route_table_association" "bt01_public_subnet_association" {
  subnet_id      = aws_subnet.bt01_public_subnet.id
  route_table_id = aws_route_table.bt01_route_table.id
}

resource "aws_route_table_association" "bt01_private_subnet_association" {
  subnet_id      = aws_subnet.bt01_private_subnet.id
  route_table_id = aws_route_table.bt01_route_table.id
}
```

> Lab note: because the second subnet uses a route to the Internet
> Gateway, it is not technically private. This is a deliberate lab
> simplification.

Run:

``` powershell
terraform fmt
terraform validate
terraform plan
terraform apply
```

------------------------------------------------------------------------

# PART 7 --- Aurora PostgreSQL

## Step 9 --- DB subnet group

Create `aurora.tf`:

``` hcl
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
```

------------------------------------------------------------------------

# Step 10 --- Security group

``` hcl
resource "aws_security_group" "bt01_aurora" {
  name        = "bt01-aurora-sg"
  description = "Security group for Aurora PostgreSQL"
  vpc_id      = aws_vpc.bt01_vpc.id

  ingress {
    description = "PostgreSQL"
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
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
```

> Lab only: `0.0.0.0/0` on PostgreSQL is intentionally simple. A
> production design should restrict access.

------------------------------------------------------------------------

# Step 11 --- Aurora cluster

``` hcl
resource "aws_rds_cluster" "bt01_aurora" {
  cluster_identifier = "bt01-aurora"
  engine             = "aurora-postgresql"

  master_username = "postgres"
  master_password = "postgres"

  database_name         = "paylite"
  db_subnet_group_name  = aws_db_subnet_group.bt01_aurora.name
  vpc_security_group_ids = [aws_security_group.bt01_aurora.id]

  skip_final_snapshot = true

  tags = {
    Name = "bt01-aurora"
  }
}
```

------------------------------------------------------------------------

# Step 12 --- Aurora instance

``` hcl
resource "aws_rds_cluster_instance" "bt01_aurora" {
  identifier         = "bt01-aurora-instance-1"
  cluster_identifier = aws_rds_cluster.bt01_aurora.id

  instance_class = "db.t3.medium"
  engine         = "aurora-postgresql"

  tags = {
    Name = "bt01-aurora-instance-1"
  }
}
```

Run:

``` powershell
terraform fmt
terraform validate
terraform plan
```

Read the plan carefully.

Then:

``` powershell
terraform apply
```

------------------------------------------------------------------------

# Step 13 --- Aurora outputs

Add:

``` hcl
output "aurora_endpoint" {
  description = "Aurora writer endpoint"
  value       = aws_rds_cluster.bt01_aurora.endpoint
}

output "aurora_reader_endpoint" {
  description = "Aurora reader endpoint"
  value       = aws_rds_cluster.bt01_aurora.reader_endpoint
}

output "aurora_port" {
  description = "Aurora PostgreSQL port"
  value       = aws_rds_cluster.bt01_aurora.port
}
```

Run:

``` powershell
terraform output
```

------------------------------------------------------------------------

# Step 14 --- Verify Terraform

``` powershell
terraform state list
terraform show
terraform output
```

Expected resource types include:

``` text
aws_vpc
aws_internet_gateway
aws_subnet
aws_route_table
aws_route_table_association
aws_db_subnet_group
aws_security_group
aws_rds_cluster
aws_rds_cluster_instance
```

------------------------------------------------------------------------

# Step 15 --- Git checkpoint

At this milestone:

``` powershell
git status
git add .
git commit -m "Build Aurora PostgreSQL infrastructure with Terraform"
git push origin main
```

------------------------------------------------------------------------

# PART 8 --- EC2 Automation Server

## Step 16 --- Connect to EC2

The EC2 server becomes the automation/control point.

Conceptually:

``` text
GitHub
   |
   v
EC2
 |
 +-- Ansible
 +-- Liquibase
 +-- Jenkins
 +-- monitoring2.sh
 |
 v
PostgreSQL
```

------------------------------------------------------------------------

# PART 9 --- Install Ansible

## Step 17 --- Install Ansible

On Amazon Linux:

``` bash
sudo dnf install ansible-core -y
```

Verify:

``` bash
ansible --version
```

------------------------------------------------------------------------

# PART 10 --- PostgreSQL Python Dependency

## Step 18 --- Install psycopg support

``` bash
sudo dnf install python3-psycopg2 -y
```

Install the PostgreSQL collection:

``` bash
ansible-galaxy collection install community.postgresql
```

Verify:

``` bash
ansible-galaxy collection list
```

------------------------------------------------------------------------

# PART 11 --- Pull the Git Repository

## Step 19 --- Clone

``` bash
cd /opt
git clone https://github.com/postgreshelp/ansible-postgresql.git
cd /opt/ansible-postgresql
```

For future updates:

``` bash
cd /opt/ansible-postgresql
git pull origin main
```

Git is now the source of truth for the project files.

------------------------------------------------------------------------

# PART 12 --- Ansible Configuration

## Step 20 --- ansible.cfg

Create:

``` ini
[defaults]
inventory = ./inventory/hosts
host_key_checking = False
```

------------------------------------------------------------------------

# Step 21 --- Inventory

Create `inventory/hosts`:

``` ini
[postgresql]
localhost ansible_connection=local
```

Test:

``` bash
ansible all --list-hosts
```

------------------------------------------------------------------------

# Step 22 --- PostgreSQL variables

Create:

`inventory/group_vars/postgresql.yml`

``` yaml
---
postgresql_host: "YOUR_DATABASE_ENDPOINT"
postgresql_port: 5432
postgresql_admin_user: "postgres"
postgresql_admin_password: "postgres"
postgresql_database: "postgres"
```

Important:

``` text
group_vars/
    |
    v
postgresql.yml
    |
    v
[postgresql]
```

Ansible automatically loads variables matching the inventory group.

For real environments, move passwords to Ansible Vault or another
secret-management mechanism.

------------------------------------------------------------------------

# PART 13 --- First PostgreSQL Automation

## Step 23 --- Create application role

`playbooks/postgresql_role.yml`

``` yaml
---
- name: Manage PostgreSQL roles
  hosts: postgresql
  gather_facts: false

  tasks:
    - name: Create PayLite application role
      community.postgresql.postgresql_user:
        name: paylite_app
        password: "PayliteApp123!"
        state: present
        login_host: "{{ postgresql_host }}"
        login_port: "{{ postgresql_port }}"
        login_user: "{{ postgresql_admin_user }}"
        login_password: "{{ postgresql_admin_password }}"
        login_db: "{{ postgresql_database }}"
```

Run:

``` bash
ansible-playbook playbooks/postgresql_role.yml
```

Expected:

``` text
changed: [localhost]
```

Run it again.

The second run should normally show the resource as already present
rather than recreating it.

This demonstrates the importance of **idempotency**.

------------------------------------------------------------------------

# PART 14 --- PostgreSQL Administration

## Step 24 --- Version check

Create `playbooks/postgresql_admin.yml`:

``` yaml
---
- name: PostgreSQL administrative tasks
  hosts: postgresql
  gather_facts: false

  tasks:
    - name: Check PostgreSQL version
      community.postgresql.postgresql_query:
        query: "SELECT version();"
        login_host: "{{ postgresql_host }}"
        login_port: "{{ postgresql_port }}"
        login_user: "{{ postgresql_admin_user }}"
        login_password: "{{ postgresql_admin_password }}"
        login_db: "{{ postgresql_database }}"
      register: postgres_version

    - name: Display PostgreSQL version
      ansible.builtin.debug:
        var: postgres_version.query_result
```

Run:

``` bash
ansible-playbook playbooks/postgresql_admin.yml
```

------------------------------------------------------------------------

# Step 25 --- Expand administration

Add tasks for:

``` text
current_database()
database sizes
connections
active sessions
replication information
configuration values
roles
privileges
```

The objective is not to turn Ansible into a SQL editor.

The objective is to automate repeatable DBA operations.

------------------------------------------------------------------------

# Step 26 --- Git checkpoint

``` bash
git add .
git commit -m "Add Ansible PostgreSQL administration"
git push origin main
```

------------------------------------------------------------------------

# PART 15 --- Liquibase

## Step 27 --- Install Java

Verify Java:

``` bash
java -version
```

Liquibase requires Java.

------------------------------------------------------------------------

# Step 28 --- Install Liquibase

Install Liquibase under:

``` text
/opt/liquibase
```

Verify:

``` bash
liquibase --version
```

------------------------------------------------------------------------

# Step 29 --- PostgreSQL JDBC driver

Create the library directory if required:

``` bash
mkdir -p /opt/liquibase/lib
```

Download the PostgreSQL JDBC driver:

``` bash
curl -L -o /opt/liquibase/lib/postgresql.jar \
https://jdbc.postgresql.org/download/postgresql-42.7.8.jar
```

Verify:

``` bash
ls -l /opt/liquibase/lib/
```

------------------------------------------------------------------------

# Step 30 --- Liquibase project

Inside the Git repository:

``` text
liquibase/
├── liquibase.properties
├── db.changelog-master.yaml
└── changelog/
    ├── 001-create-schema.sql
    └── 002-create-table.sql
```

------------------------------------------------------------------------

# Step 31 --- Liquibase properties

`liquibase/liquibase.properties`

``` properties
url=jdbc:postgresql://YOUR_DATABASE_ENDPOINT:5432/postgres
username=postgres
password=postgres
changelog-file=db.changelog-master.yaml
```

Use `=` syntax in this properties file.

Do not commit real passwords.

------------------------------------------------------------------------

# Step 32 --- First changeset

`liquibase/changelog/001-create-schema.sql`

``` sql
--liquibase formatted sql

--changeset paylite:001
CREATE SCHEMA IF NOT EXISTS paylite;
```

------------------------------------------------------------------------

# Step 33 --- Second changeset

`liquibase/changelog/002-create-table.sql`

``` sql
--liquibase formatted sql

--changeset paylite:002
CREATE TABLE IF NOT EXISTS paylite.employee (
    employee_id BIGSERIAL PRIMARY KEY,
    employee_name VARCHAR(100) NOT NULL,
    department VARCHAR(100),
    salary NUMERIC(12,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

------------------------------------------------------------------------

# Step 34 --- Master changelog

`liquibase/db.changelog-master.yaml`

``` yaml
databaseChangeLog:
  - include:
      file: changelog/001-create-schema.sql

  - include:
      file: changelog/002-create-table.sql
```

------------------------------------------------------------------------

# Step 35 --- Validate

Run from the Liquibase directory:

``` bash
cd /opt/ansible-postgresql/liquibase
liquibase validate
```

------------------------------------------------------------------------

# Step 36 --- Check status

``` bash
liquibase status
```

------------------------------------------------------------------------

# Step 37 --- Apply changes

``` bash
liquibase update
```

Verify:

``` bash
liquibase status
```

Then verify from PostgreSQL:

``` sql
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_schema = 'paylite'
  AND table_name = 'employee';
```

------------------------------------------------------------------------

# Step 38 --- Understand the result

Liquibase now knows:

``` text
001 → executed
002 → executed
```

The database contains Liquibase tracking information.

The next time `liquibase update` runs, already-executed changesets are
not simply executed again.

------------------------------------------------------------------------

# Step 39 --- Git checkpoint

``` bash
git add .
git commit -m "Add Liquibase database changelogs"
git push origin main
```

------------------------------------------------------------------------

# PART 16 --- Ansible + Liquibase

## Step 40 --- Why combine them?

Ansible can execute the Liquibase command.

The responsibilities remain separate:

``` text
Ansible
  |
  | execute/manage
  v
Liquibase
  |
  | database change management
  v
PostgreSQL
```

------------------------------------------------------------------------

# Step 41 --- Liquibase deployment playbook

Create:

`playbooks/liquibase_deploy.yml`

``` yaml
---
- name: Deploy database changes using Liquibase
  hosts: postgresql
  gather_facts: false

  tasks:
    - name: Check Liquibase installation
      ansible.builtin.command:
        cmd: liquibase --version
      register: liquibase_version
      changed_when: false

    - name: Display Liquibase version
      ansible.builtin.debug:
        var: liquibase_version.stdout_lines

    - name: Validate Liquibase changelog
      ansible.builtin.command:
        cmd: liquibase validate
        chdir: "{{ playbook_dir }}/../liquibase"
      register: liquibase_validate
      changed_when: false

    - name: Display Liquibase validation
      ansible.builtin.debug:
        var: liquibase_validate.stdout_lines

    - name: Check Liquibase status
      ansible.builtin.command:
        cmd: liquibase status
        chdir: "{{ playbook_dir }}/../liquibase"
      register: liquibase_status
      changed_when: false

    - name: Display Liquibase status
      ansible.builtin.debug:
        var: liquibase_status.stdout_lines

    - name: Apply Liquibase changes
      ansible.builtin.command:
        cmd: liquibase update
        chdir: "{{ playbook_dir }}/../liquibase"
      register: liquibase_update

    - name: Display Liquibase update result
      ansible.builtin.debug:
        var: liquibase_update.stdout_lines

    - name: Verify PayLite employee table
      community.postgresql.postgresql_query:
        query: |
          SELECT
              table_schema,
              table_name
          FROM information_schema.tables
          WHERE table_schema = 'paylite'
            AND table_name = 'employee';
        login_host: "{{ postgresql_host }}"
        login_port: "{{ postgresql_port }}"
        login_user: "{{ postgresql_admin_user }}"
        login_password: "{{ postgresql_admin_password }}"
        login_db: "{{ postgresql_database }}"
      register: employee_table

    - name: Display PayLite employee table
      ansible.builtin.debug:
        var: employee_table.query_result
```

Run:

``` bash
ansible-playbook playbooks/liquibase_deploy.yml
```

The workflow is now:

``` text
Ansible
   |
   v
Validate Liquibase
   |
   v
Check status
   |
   v
Liquibase update
   |
   v
Verify PostgreSQL
```

------------------------------------------------------------------------

# PART 17 --- Jenkins

## Step 42 --- Install Jenkins

Install Jenkins on the EC2 automation server.

Verify:

``` bash
systemctl status jenkins
```

------------------------------------------------------------------------

# Step 43 --- Jenkins concept

At this stage, Jenkins should remain deliberately small.

The goal is:

``` text
Git
 ↓
Jenkins
 ↓
Ansible
 ↓
Liquibase
 ↓
PostgreSQL
```

Do not turn this module into Jenkins administration training.

------------------------------------------------------------------------

# Step 44 --- Jenkins executes Ansible

A simple shell command:

``` bash
cd /opt/ansible-postgresql
ansible-playbook playbooks/liquibase_deploy.yml
```

If Jenkins needs the project configuration explicitly:

``` bash
sudo -u jenkins env \
ANSIBLE_CONFIG=/opt/ansible-postgresql/ansible.cfg \
ansible-playbook \
/opt/ansible-postgresql/playbooks/liquibase_deploy.yml
```

------------------------------------------------------------------------

# Step 45 --- Jenkins pipeline

Create `Jenkinsfile`:

``` groovy
pipeline {
    agent any

    stages {

        stage('Verify Tools') {
            steps {
                sh 'terraform version || true'
                sh 'ansible --version'
                sh 'liquibase --version'
            }
        }

        stage('PostgreSQL Administration') {
            steps {
                sh '''
                    cd /opt/ansible-postgresql
                    ansible-playbook playbooks/postgresql_admin.yml
                '''
            }
        }

        stage('Database Deployment') {
            steps {
                sh '''
                    cd /opt/ansible-postgresql
                    ansible-playbook playbooks/liquibase_deploy.yml
                '''
            }
        }

        stage('Monitoring') {
            steps {
                sh '''
                    cd /opt/ansible-postgresql
                    ./monitoring2.sh
                '''
            }
        }
    }
}
```

The pipeline is intentionally simple.

------------------------------------------------------------------------

# PART 18 --- Final Git Workflow

## Step 46 --- Developer changes a database

Example:

``` text
003-create-index.sql
```

Commit:

``` bash
git add .
git commit -m "Add employee salary index"
git push origin main
```

------------------------------------------------------------------------

# Step 47 --- Jenkins receives the change

Conceptually:

``` text
Developer
   |
   | git push
   v
GitHub
   |
   v
Jenkins
   |
   +--> checkout
   |
   +--> validate
   |
   +--> Ansible
   |
   +--> Liquibase
   |
   +--> monitoring
```

------------------------------------------------------------------------

# PART 19 --- Environment Promotion

The learning architecture can be expanded from one database to three:

``` text
DEV
TEST
PROD
```

The same versioned changelog can be promoted:

``` text
Git
 |
 v
Liquibase
 |
 v
DEV
 |
 | approval
 v
TEST
 |
 | approval
 v
PROD
```

At this stage, introduce:

-   separate endpoints;
-   separate credentials;
-   environment-specific variables;
-   promotion rules;
-   deployment status;
-   failure handling.

------------------------------------------------------------------------

# PART 20 --- Final End-to-End Lab

## Step 48 --- Full sequence

Run the project as one workflow:

``` text
1. Developer changes Terraform / Ansible / Liquibase files
2. git add
3. git commit
4. git push
5. EC2 pulls repository
6. Jenkins starts
7. Terraform manages infrastructure
8. Ansible manages PostgreSQL operations
9. Liquibase validates changelog
10. Liquibase applies schema changes
11. PostgreSQL is verified
12. monitoring2.sh runs
13. Jenkins reports success/failure
```

------------------------------------------------------------------------

# PART 21 --- Failure Exercise

Introduce a controlled failure.

Examples:

``` text
Terraform configuration error
Liquibase SQL error
Wrong database endpoint
PostgreSQL authentication failure
Ansible task failure
Jenkins stage failure
```

Observe:

``` text
Where did it fail?
Why did it fail?
Which tool reported it?
What changed?
What should happen next?
```

------------------------------------------------------------------------

# PART 22 --- Final Architecture

``` text
                         GITHUB
                    Source of Truth
                          |
                       git push
                          |
                          v
             +--------------------------+
             |       EC2 SERVER         |
             |                          |
             |       JENKINS            |
             |          |               |
             |   +------+------+        |
             |   |      |      |        |
             | Terraform Ansible Liquibase
             |   |      |      |        |
             +---|------|------|--------+
                 |      |      |
                 v      v      v
              AWS     PostgreSQL Schema
          Infrastructure Operations Changes
                         |
                         v
                  +-------------+
                  | PostgreSQL  |
                  |             |
                  | DEV         |
                  | TEST        |
                  | PROD        |
                  +-------------+
                         |
                         v
                  monitoring2.sh
```

------------------------------------------------------------------------

# Final Learning Outcome

The learner should be able to explain:

``` text
Git
 ↓
Version and collaborate

Terraform
 ↓
Provision/manage infrastructure

Ansible
 ↓
Automate PostgreSQL operations

Liquibase
 ↓
Manage versioned database changes

Jenkins
 ↓
Orchestrate the workflow
```

The objective is not to memorize five tools.

The objective is to understand **why each tool exists, where it fits,
how the tools interact, and how to operate the complete PostgreSQL
automation workflow safely**.
