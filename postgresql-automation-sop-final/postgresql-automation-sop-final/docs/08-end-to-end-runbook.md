# 08 - End-to-End Runbook

## Verify Ansible

```bash
cd ~/ansible-postgresql
ansible --version
ansible postgresql -m ping
```

## Verify PostgreSQL

```bash
psql -h <RDS_ENDPOINT> -U postgres -d postgres
```

Exit:

```sql
\q
```

## Run administration

```bash
ansible-playbook playbooks/postgresql_admin_full.yml
```

## Verify Liquibase

```bash
cd liquibase
liquibase --version
liquibase validate
liquibase status
cd ..
```

## Run Liquibase through Ansible

```bash
ansible-playbook playbooks/liquibase_deploy.yml
```

## Verify schema

```bash
psql -h <RDS_ENDPOINT> -U postgres -d postgres
```

```sql
\dn
\dt paylite.*
```

Expected:

```text
paylite.employee
```

## Run Jenkins

Create/run a Jenkins Pipeline using `jenkins/Jenkinsfile`.

The automated workflow is:

```text
Jenkins
  |
  v
Ansible
  |
  +--> PostgreSQL administration
  |
  +--> Liquibase validate
  |
  +--> Liquibase status
  |
  +--> Liquibase update
  |
  v
PostgreSQL
```

Run the pipeline again to demonstrate idempotency.

For a new database change, add a new numbered changeset and include it in the master changelog.
