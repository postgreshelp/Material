# 03 - PostgreSQL Administration

## Configure connection variables

Edit:

```bash
vi inventory/group_vars/postgresql.yml
```

Set the real endpoint/password, keeping the structure:

```yaml
---
postgresql_host: "<RDS_ENDPOINT>"
postgresql_port: 5432
postgresql_admin_user: "postgres"
postgresql_admin_password: "<RDS_PASSWORD>"
postgresql_database: "postgres"
```

## Verify connectivity

```bash
psql -h <RDS_ENDPOINT> -U postgres -d postgres
```

Inside psql:

```sql
SELECT version();
\du
```

Exit:

```sql
\q
```

## Manage application role

```bash
ansible-playbook playbooks/postgresql_role.yml
```

Run it a second time to demonstrate idempotency.

## Basic administration

```bash
ansible-playbook playbooks/postgresql_admin.yml
```

## Full administration lab

```bash
ansible-playbook playbooks/postgresql_admin_full.yml
```

The full playbook covers version, database size, connections, active sessions, replication status, reporting role, CONNECT privilege, ANALYZE and selected configuration values.

Ansible operates against PostgreSQL; it does not attempt OS-level SSH configuration of managed RDS/Aurora.
