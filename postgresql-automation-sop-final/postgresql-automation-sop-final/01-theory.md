# PostgreSQL Automation Project --- Theory

## 1. Big Picture

This project uses five technologies, each with a different
responsibility:

  Technology   Primary responsibility
  ------------ --------------------------------------------
  Git          Version control and source of truth
  Terraform    Provision and manage infrastructure
  Ansible      Configure and operate systems / PostgreSQL
  Liquibase    Manage database schema changes
  Jenkins      Orchestrate the workflow

The key idea is **separation of responsibility**.

``` mermaid
flowchart LR
    G[Git\nSource of Truth]
    T[Terraform\nInfrastructure]
    A[Ansible\nOperations]
    L[Liquibase\nDatabase Changes]
    J[Jenkins\nOrchestration]
    P[(PostgreSQL)]

    G --> T
    G --> A
    G --> L
    G --> J
    T --> P
    A --> P
    L --> P
    J --> T
    J --> A
    J --> L
```

------------------------------------------------------------------------

# 2. Git

## What is Git?

Git is a **version control system**.

It records changes to files over time so that we can:

-   see what changed;
-   compare versions;
-   go back to an earlier version;
-   work with branches;
-   collaborate;
-   connect code/configuration to deployments.

For this project, Git is the **thread connecting all technologies**.

## Minimal workflow

``` text
Working files
    |
    v
git add
    |
    v
git commit
    |
    v
git push
    |
    v
GitHub
```

### Bare-minimum commands

``` bash
git init
git add .
git commit -m "Add Terraform configuration"
git push
```

## Git in this project

Git stores:

``` text
Terraform files
Ansible inventory/playbooks
Liquibase changelogs
Jenkins pipeline files
Documentation
```

Git does **not** create AWS resources, configure PostgreSQL, or execute
database migrations by itself.

------------------------------------------------------------------------

# 3. Terraform

## What is Terraform?

Terraform is an **infrastructure-as-code tool**.

It lets us describe the infrastructure we want in configuration files
and then create or modify that infrastructure.

``` text
Terraform configuration
        |
        v
Desired infrastructure
        |
        v
terraform plan
        |
        v
terraform apply
        |
        v
AWS infrastructure
```

## Problem Terraform solves

Without Terraform:

``` text
AWS Console
   ↓
click VPC
click subnet
click route table
click security group
click database
...
```

With Terraform:

``` text
.tf files
   ↓
terraform plan
   ↓
terraform apply
```

The configuration becomes repeatable and reviewable.

## Core concepts

### Provider

Connects Terraform to a platform such as AWS.

``` hcl
provider "aws" {
  region = "us-east-1"
}
```

### Resource

Declares infrastructure Terraform should manage.

``` hcl
resource "aws_vpc" "example" {
  cidr_block = "10.0.0.0/16"
}
```

### Data source

Reads an existing object.

``` hcl
data "aws_vpc" "default" {
  default = true
}
```

### Variable

Makes configuration reusable.

``` hcl
variable "region" {
  type = string
}
```

### Output

Displays a useful value.

``` hcl
output "vpc_id" {
  value = aws_vpc.example.id
}
```

### State

Terraform maintains state so it can understand the relationship between
configuration and infrastructure.

``` text
Configuration
     |
     +------+
     |      |
     v      v
  State   Real AWS
     |      |
     +------+
        |
        v
      Plan
```

## Terraform lifecycle

``` text
Write
  ↓
init
  ↓
validate
  ↓
plan
  ↓
apply
  ↓
state
```

## Important mental model

`terraform apply` does not simply mean "create everything."

It means:

> Make real infrastructure match the Terraform configuration.

That may involve:

-   creating;
-   modifying;
-   replacing;
-   or destroying resources.

Always inspect `terraform plan` before applying.

------------------------------------------------------------------------

# 4. Ansible

## What is Ansible?

Ansible is an **automation and configuration management tool**.

It executes tasks against managed targets.

Typical examples:

-   install packages;
-   configure services;
-   create users;
-   run commands;
-   manage PostgreSQL roles;
-   execute PostgreSQL queries.

## Basic model

``` text
Ansible control/execution node
            |
            v
       Inventory
            |
            v
       Playbook
            |
            v
          Tasks
            |
            v
        Target
```

## Minimal playbook

``` yaml
---
- name: Example
  hosts: all
  gather_facts: false

  tasks:
    - name: Show message
      ansible.builtin.debug:
        msg: "Hello"
```

## Important terms

### Inventory

Defines the hosts/groups Ansible operates against.

``` ini
[postgresql]
localhost ansible_connection=local
```

### Playbook

Defines the automation workflow.

``` yaml
- name: Manage PostgreSQL
  hosts: postgresql
  tasks:
    ...
```

### Task

One unit of work.

``` yaml
- name: Create role
  community.postgresql.postgresql_user:
    ...
```

### Module

The code that performs a specific operation.

Examples:

``` text
ansible.builtin.command
community.postgresql.postgresql_user
community.postgresql.postgresql_query
```

## Ansible + RDS/Aurora

An important distinction:

``` text
                 EC2
          Ansible execution
                 |
                 | PostgreSQL connection
                 | TCP 5432
                 v
        RDS / Aurora PostgreSQL
```

The RDS/Aurora database is **not an SSH server that Ansible manages as
an operating-system host**.

In this lab:

``` ini
[postgresql]
localhost ansible_connection=local
```

means Ansible executes on the EC2 machine.

The PostgreSQL module then connects to the remote database using:

``` yaml
login_host
login_port
login_user
login_password
login_db
```

------------------------------------------------------------------------

# 5. Liquibase

## What is Liquibase?

Liquibase is a **database change management tool**.

It manages database changes as versioned changesets/changelogs.

The important difference is:

``` text
SQL alone
    =
execute a database change

Liquibase
    =
manage the lifecycle of database changes
```

That lifecycle can include:

``` text
Define
  ↓
Track
  ↓
Validate
  ↓
Deploy
  ↓
Record history
  ↓
Manage future changes
```

## Minimal formatted SQL changelog

``` sql
--liquibase formatted sql

--changeset paylite:001
CREATE SCHEMA IF NOT EXISTS paylite;
```

Another change:

``` sql
--liquibase formatted sql

--changeset paylite:002
CREATE TABLE IF NOT EXISTS paylite.employee (
    employee_id BIGSERIAL PRIMARY KEY,
    employee_name VARCHAR(100) NOT NULL
);
```

## Master changelog

A master file defines the order of changes.

``` yaml
databaseChangeLog:
  - include:
      file: changelog/001-create-schema.sql

  - include:
      file: changelog/002-create-table.sql
```

## Liquibase commands

``` bash
liquibase validate
liquibase status
liquibase update
```

## Liquibase history

Liquibase maintains database-side tracking information.

Conceptually:

``` text
001  CREATE SCHEMA
002  CREATE TABLE
003  ALTER TABLE
004  CREATE INDEX
```

This is why Liquibase is useful when many changes and environments must
be controlled.

## Rollback clarification

Liquibase rollback is **not a substitute for backup/PITR**.

If a changeset performs a destructive data operation, Liquibase cannot
magically reconstruct deleted data unless a valid inverse operation was
explicitly defined and the required information still exists.

``` text
Liquibase rollback
      ≠
Data recovery
```

Use backup/PITR for actual data recovery.

------------------------------------------------------------------------

# 6. Jenkins

## What is Jenkins?

Jenkins is an **automation/orchestration server**.

It can execute a sequence of steps automatically.

For this project Jenkins is the **orchestrator**, not the tool that
replaces Terraform, Ansible, or Liquibase.

``` text
Git change
    |
    v
 Jenkins
    |
    +--> Terraform
    |
    +--> Ansible
    |
    +--> Liquibase
    |
    +--> monitoring2.sh
```

## Minimal pipeline concept

``` text
Pipeline
   |
   +-- Stage 1
   |
   +-- Stage 2
   |
   +-- Stage 3
```

A stage contains steps.

Example:

``` groovy
pipeline {
    agent any

    stages {
        stage('Verify') {
            steps {
                sh 'terraform version'
            }
        }

        stage('Deploy') {
            steps {
                sh 'ansible-playbook playbooks/postgresql_admin.yml'
            }
        }
    }
}
```

The exact Jenkins administration is intentionally kept small in this
project.

------------------------------------------------------------------------

# 7. Why These Five Together?

Each tool answers a different question.

``` text
What infrastructure do I need?
        |
        v
     Terraform

How do I configure/operate it?
        |
        v
      Ansible

How do I manage database changes?
        |
        v
    Liquibase

How do I version everything?
        |
        v
       Git

How do I automate the sequence?
        |
        v
      Jenkins
```

## Responsibility map

  Question                                               Tool
  ------------------------------------------------------ -----------
  Where should the infrastructure exist?                 Terraform
  How should the server/database be configured?          Ansible
  Which schema changes should be deployed?               Liquibase
  Which version of the code/configuration is approved?   Git
  What should run, and in what order?                    Jenkins

------------------------------------------------------------------------

# 8. What Each Tool Does NOT Mean

  Tool        Do not teach it as
  ----------- ---------------------------------------------
  Git         Deployment engine
  Terraform   PostgreSQL schema migration tool
  Ansible     Dedicated database change-management system
  Liquibase   Backup/PITR system
  Jenkins     Replacement for every other tool

The tools can overlap technically. The table represents their **primary
responsibility in this project**.

------------------------------------------------------------------------

# 9. Project Mental Model

``` text
                 GIT / GITHUB
                Source of Truth
                     |
        +------------+-------------+
        |            |             |
        v            v             v
    Terraform     Ansible      Liquibase
        |            |             |
        v            v             v
   AWS Network   PostgreSQL     DB Schema
        |         Operations      Changes
        |            |             |
        +------------+-------------+
                     |
                     v
              PostgreSQL
                     ^
                     |
                  Jenkins
              Orchestration
```

The project therefore teaches a complete delivery chain without making
any one tool responsible for everything.
