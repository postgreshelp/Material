# Ansible for PostgreSQL DBAs

## 1. What is Ansible?

Ansible is an automation/orchestration tool. A playbook contains plays and tasks; tasks call modules. Most Ansible modules aim for idempotent desired-state behavior, although not every module/action is idempotent. citeturn0search7

DBA mental model:

> **"How do I make a repetitive operational procedure executable, repeatable and consistent?"**

---

# 2. DBA nomenclature

## Control node

The machine from which Ansible runs.

In the lab:

```text
Amazon Linux EC2
```

## Managed node

A machine Ansible connects to.

For PostgreSQL DBAs, this needs clarification:

- For a normal PostgreSQL VM, Ansible may manage the OS and PostgreSQL server.
- For managed RDS/Aurora PostgreSQL, Ansible generally should operate through PostgreSQL/database APIs/modules rather than treating the database as an SSH-managed Linux server.

## Inventory

Defines hosts/groups.

```ini
[postgresql]
localhost ansible_connection=local
```

## Group

A logical collection of hosts.

```ini
[postgresql]
...
```

## Playbook

YAML automation definition.

```yaml
- name: PostgreSQL administration
  hosts: postgresql
  tasks:
    ...
```

## Play

A set of tasks applied to selected hosts.

## Task

One automation action.

## Module

The implementation used by a task.

Examples:

```text
community.postgresql.postgresql_user
community.postgresql.postgresql_query
```

## Collection

A packaged group of Ansible content.

Example:

```text
community.postgresql
```

## Role

A reusable Ansible structure for packaging tasks, handlers, templates, variables and defaults.

## Variable

Data passed to tasks.

```yaml
postgresql_host: "<RDS_ENDPOINT>"
```

## Idempotency

Running the same desired-state automation repeatedly should result in the same final state rather than repeatedly changing it. citeturn0search7

---

# 3. Important commands

## Version

```bash
ansible --version
```

## Inventory graph

```bash
ansible-inventory --graph
```

## List inventory

```bash
ansible-inventory --list
```

## Ping

```bash
ansible postgresql -m ping
```

## List collection

```bash
ansible-galaxy collection list
```

## Install PostgreSQL collection

```bash
ansible-galaxy collection install community.postgresql
```

## Run playbook

```bash
ansible-playbook playbooks/postgresql_admin.yml
```

## Verbose

```bash
ansible-playbook playbooks/postgresql_admin.yml -v
```

More detail:

```bash
ansible-playbook playbooks/postgresql_admin.yml -vvv
```

## Check mode

```bash
ansible-playbook playbooks/postgresql_admin.yml --check
```

Check mode can preview changes for modules that support it. citeturn0search7

## Limit

```bash
ansible-playbook site.yml --limit postgresql
```

## Syntax check

```bash
ansible-playbook playbooks/postgresql_admin.yml --syntax-check
```

---

# 4. PostgreSQL DBA use cases

## User/role management

```yaml
community.postgresql.postgresql_user:
```

Use for:

- application roles
- reporting roles
- password rotation workflows
- controlled role state

## Grants

```yaml
community.postgresql.postgresql_privs:
```

Use for:

- CONNECT
- SELECT
- INSERT
- UPDATE
- schema/object privileges

## Query execution

```yaml
community.postgresql.postgresql_query:
```

Use for:

- health checks
- session checks
- metadata
- operational SQL
- verification

---

# 5. Where Ansible fits in production

```text
Incident/runbook
       |
       v
Ansible playbook
       |
       +--> PostgreSQL module
       |
       +--> SQL
       |
       v
PostgreSQL
```

---

# 6. Real production incidents

## Incident: Application account missing

Symptom:

```text
Application cannot authenticate to PostgreSQL.
```

DBA determines the role should exist.

Instead of manually recreating it on 20 databases:

```text
Ansible inventory
        |
        v
postgresql_user
        |
        v
all required databases
```

Run:

```bash
ansible-playbook playbooks/postgresql_role.yml
```

---

## Incident: Reporting role lost CONNECT

Symptom:

```text
Reporting application receives permission denied / connection authorization error.
```

Ansible can restore the approved privilege consistently:

```yaml
community.postgresql.postgresql_privs:
  type: database
  database: "{{ postgresql_database }}"
  roles: paylite_reporting
  privs: CONNECT
  state: present
```

---

## Incident: Need active-session inventory

Instead of asking every DBA to execute different SQL:

```bash
ansible-playbook playbooks/postgresql_admin.yml
```

The playbook can return:

```text
pid
user
database
state
query
```

This makes the runbook repeatable.

---

## Incident: Need ANALYZE after a controlled data-load operation

If the operational procedure is approved and repeatable:

```text
Ansible
  |
  +--> ANALYZE
  |
  +--> verification query
```

This is operational automation, not schema versioning.

---

## Incident: Same emergency procedure across 50 PostgreSQL servers

Example:

```text
Check connections
Check long-running sessions
Collect configuration
Collect replication status
Run approved remediation
```

Ansible is valuable because the procedure becomes one tested playbook rather than a collection of shell commands copied between servers.

---

# 7. Ansible vs SQL

| Requirement | Use |
|---|---|
| One emergency query right now | SQL/psql |
| Repeat operation on many DBs | Ansible |
| Ensure role exists | Ansible |
| Inspect current sessions | SQL or Ansible |
| Schema deployment | Liquibase |
| AWS infrastructure | Terraform |
| Pipeline orchestration | Jenkins |

---

# 8. Production safety

Avoid putting real passwords in:

```text
inventory/group_vars/postgresql.yml
```

Use:

- Ansible Vault
- Jenkins credentials
- external secret manager
- cloud secret services

The lab can use placeholders for teaching.

---

# 9. Golden rule

```text
Ansible = repeatable operational DBA automation
```

It should not become a replacement for:

```text
Terraform -> infrastructure
Liquibase -> schema changes
Jenkins -> orchestration
PostgreSQL -> database engine
```
