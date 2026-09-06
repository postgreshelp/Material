# Production Placement Guide

## One picture

```text
                     ┌─────────────────────┐
                     │       Git           │
                     └──────────┬──────────┘
                                |
             ┌──────────────────┼──────────────────┐
             |                  |                  |
             v                  v                  v
        Terraform           Ansible           Liquibase
             |                  |                  |
             v                  v                  v
       AWS platform       DBA operations      DB schema
             |                  |                  |
             └──────────────────┴──────────┬───────┘
                                          |
                                          v
                                      PostgreSQL
                                          ^
                                          |
                                      Jenkins
                              orchestration / approval
```

## Ownership boundaries

### Terraform owns

```text
VPC
subnets
security groups
RDS/Aurora
DB subnet groups
parameter groups
infrastructure IAM
infrastructure monitoring
```

### Ansible owns

```text
repeatable DBA operations
roles/users
grants
health checks
operational SQL
runbooks
multi-server DBA procedures
```

### Liquibase owns

```text
schema
tables
columns
indexes
views
constraints
database release history
```

### Jenkins owns

```text
pipeline
credentials integration
approval
scheduling
environment promotion
logs
build history
```

## Anti-overlap rule

Do not allow two systems to own the same lifecycle object.

Bad:

```text
Terraform creates application table
+
Liquibase also manages application table
```

Good:

```text
Terraform -> RDS infrastructure
Liquibase -> application schema
```

Bad:

```text
Jenkinsfile contains hundreds of SQL statements
```

Good:

```text
Jenkins -> Liquibase
```

Bad:

```text
Ansible playbook embeds all application schema creation
```

Good:

```text
Ansible -> operational DBA tasks
Liquibase -> schema
```
