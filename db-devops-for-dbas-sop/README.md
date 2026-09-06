# DB Automation & CI/CD for PostgreSQL DBAs

A practical DBA-focused reference for four tools:

1. Terraform — infrastructure as code
2. Ansible — operational automation
3. Liquibase — database change management
4. Jenkins — CI/CD orchestration

The emphasis is not on learning DevOps terminology for its own sake. The emphasis is:

> **When a DBA encounters a real production problem, where does each tool fit, what should the DBA change, what should the DBA NOT change, and how can the work be made repeatable?**

## End-to-end production model

```text
Git
 |
 +--> Terraform
 |      |
 |      +--> VPC / subnet / security group
 |      +--> RDS / Aurora / parameter groups
 |
 +--> Ansible
 |      |
 |      +--> DBA operational tasks
 |      +--> roles/users
 |      +--> grants
 |      +--> monitoring / checks
 |      +--> runbooks
 |
 +--> Liquibase
 |      |
 |      +--> schema changes
 |      +--> tables/indexes/views
 |      +--> release history
 |      +--> controlled rollback
 |
 +--> Jenkins
        |
        +--> validate
        +--> test
        +--> approve
        +--> execute
        +--> audit
```

## The DBA decision rule

| Production need | Primary tool |
|---|---|
| Create/modify AWS infrastructure | Terraform |
| Standardize repetitive DBA operations | Ansible |
| Deploy database schema changes | Liquibase |
| Automate the complete workflow | Jenkins |
| Kill one bad PostgreSQL session now | SQL / DBA action |
| Investigate a slow query | PostgreSQL tools / monitoring |
| Permanently change infrastructure configuration | Terraform |
| Permanently change database schema | Liquibase |

## Documentation map

- `terraform/01-terraform-for-dbas.md`
- `ansible/01-ansible-for-dbas.md`
- `liquibase/01-liquibase-for-dbas.md`
- `jenkins/01-jenkins-for-dbas.md`
- `incidents/01-realtime-dba-incidents.md`
- `cheatsheets/01-command-cheatsheet.md`

Each tool document contains definitions, nomenclature, commands, DBA use cases, production placement and incident examples.
