# 01 - Architecture and Prerequisites

## Architecture

```text
Terraform -> AWS RDS/Aurora PostgreSQL
                    |
                    v
                 Ansible
              /           \
 PostgreSQL administration  Liquibase
                              |
                              v
                         PostgreSQL schema
                              ^
                              |
                           Jenkins
```

## Responsibilities

- Terraform: AWS infrastructure only.
- Ansible: PostgreSQL administration, roles/users and orchestration.
- Liquibase: schema/database changes.
- Jenkins: automation.

## Lab prerequisites

- AWS RDS/Aurora PostgreSQL already created.
- Amazon Linux 2023 EC2 control node.
- Network access from the control node to PostgreSQL port 5432.
- RDS endpoint and admin credentials.
- Terraform is not required on the Ansible control node.

## Versions used in the lab

- Python 3.9
- Ansible Core 2.15.3
- community.postgresql 4.2.0
- PostgreSQL client 18.6
- Java 21 Amazon Corretto
- Liquibase 5.0.4
- PostgreSQL JDBC driver 42.7.8
- Jenkins compatible with Java 21
