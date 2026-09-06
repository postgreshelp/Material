# PostgreSQL Database Automation SOP

Clean lab SOP for AWS PostgreSQL automation using Terraform, Ansible, Liquibase and Jenkins.

## Tool responsibilities

| Tool | Responsibility |
|---|---|
| Terraform | AWS infrastructure |
| Ansible | PostgreSQL administration and orchestration |
| Liquibase | Database schema changes |
| Jenkins | CI/CD orchestration |

## Final flow

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

## Main SOP order

1. Architecture and prerequisites
2. Prepare Ansible control node
3. Configure Ansible
4. Verify PostgreSQL connectivity
5. PostgreSQL administration
6. Install Java 21
7. Install Liquibase and PostgreSQL JDBC driver
8. Create and test Liquibase project
9. Integrate Liquibase with Ansible
10. Install/configure Jenkins with Java 21
11. Run the Jenkins pipeline
12. Verify and test idempotency

The main documentation contains only the final repeatable procedure. Setup failures and lessons learned are in `appendix/`.

**Security:** replace `<RDS_PASSWORD>` with a secure secret. Do not commit real passwords to Git.
