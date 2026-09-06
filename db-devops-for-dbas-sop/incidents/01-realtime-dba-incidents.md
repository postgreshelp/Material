# Real-Time Production DBA Incidents: Where the Four Tools Fit

This document is deliberately incident-oriented.

## Incident 1 — RDS storage almost full

### Symptom

```text
Production PostgreSQL storage = 90%+
```

### DBA investigation

Use PostgreSQL/AWS monitoring first.

### Tool ownership

```text
Monitoring -> identify problem
PostgreSQL -> investigate usage
Terraform -> manage infrastructure change
Jenkins -> automate approved change
```

### Correct flow

```text
Incident
  |
  v
Confirm storage issue
  |
  v
Review Terraform
  |
  v
terraform plan
  |
  v
Approval
  |
  v
terraform apply
```

---

# Incident 2 — Application cannot connect

### Symptom

```text
connection refused / timeout / authorization failure
```

### Decision tree

```text
Can network reach DB?
       |
       +-- No --> AWS networking / Terraform
       |
       +-- Yes
             |
             v
Is PostgreSQL accepting connection?
             |
             v
Is user valid?
             |
             v
Is CONNECT privilege valid?
```

### Tools

- Terraform: network/security infrastructure
- Ansible: role/grant correction
- PostgreSQL: direct investigation
- Jenkins: automate the approved remediation

---

# Incident 3 — Missing application role on multiple DBs

Use Ansible.

```text
inventory
   |
   v
postgresql_user
   |
   v
multiple databases
```

Do not create the same role manually 30 times.

---

# Incident 4 — Application release requires new table

Use Liquibase.

```text
Git
 |
 v
Liquibase changeset
 |
 v
validate
 |
 v
test
 |
 v
Jenkins approval
 |
 v
update
```

Do not use Terraform.

---

# Incident 5 — Query is slow after release

### Investigation

Use:

```text
EXPLAIN
EXPLAIN ANALYZE
pg_stat_activity
pg_stat_statements
AWS/DB monitoring
```

The investigation itself is PostgreSQL work.

If the approved fix is an index:

```text
Liquibase -> create index
```

If the issue is an infrastructure resource/configuration problem:

```text
Terraform
```

If the investigation/remediation is repetitive across many databases:

```text
Ansible
```

---

# Incident 6 — Wrong AWS security rule

Use Terraform if the security group is Terraform-managed.

```text
Incident
 -> identify desired rule
 -> modify Terraform
 -> plan
 -> review
 -> apply
```

Do not leave a manual console fix permanently undocumented.

---

# Incident 7 — Existing RDS not in Terraform

Use Terraform import.

```text
Existing RDS
    |
    v
import
    |
    v
state
    |
    v
resource configuration
    |
    v
plan
```

Import brings infrastructure into Terraform management; it does not replace the need for correct resource configuration. citeturn0search5turn0search10

---

# Incident 8 — Liquibase deployment failed

First:

```bash
liquibase status
```

Then inspect:

```sql
SELECT id, author, filepath, dateexecuted
FROM databasechangelog
ORDER BY dateexecuted;
```

Determine whether:

- the changeset ran
- SQL partially executed
- a lock exists
- the change can safely be corrected
- a forward fix is safer than rollback

Do not blindly rerun production SQL.

---

# Incident 9 — Two DB deployment jobs started together

Liquibase's changelog lock is a database-level protection for concurrent Liquibase execution.

Jenkins should also prevent the process at the orchestration layer.

Correct production design:

```text
Jenkins concurrency control
        +
Liquibase database lock
```

---

# Incident 10 — Jenkins cannot deploy because Ansible collection is missing

Check as the Jenkins user:

```bash
sudo -u jenkins ansible-galaxy collection list | grep postgresql
```

Install into the Jenkins user's environment:

```bash
sudo -u jenkins ansible-galaxy collection install community.postgresql
```

The key production lesson:

> The Jenkins service account has a different runtime environment from root or your interactive login.

---

# Incident 11 — Jenkins runs with wrong Java

Check:

```bash
java -version
sudo -u jenkins java -version
sudo journalctl -u jenkins --no-pager
```

For the lab/current Jenkins baseline, use Java 21.

---

# Incident 12 — Emergency SQL fix was executed manually

This is not primarily a tool failure. It is a process problem.

After stabilization:

1. Document the emergency change.
2. Create the corresponding Liquibase changeset.
3. Ensure the repository represents the intended final schema.
4. Verify production against the changeset history.
5. Prevent future drift.

---

# Incident 13 — DBA needs the same health report every morning

Use Ansible for repeatable collection.

Example checks:

```text
PostgreSQL version
database size
connection count
active sessions
replication status
selected configuration
```

Jenkins can schedule the Ansible playbook if a CI/CD scheduler is appropriate.

---

# Incident 14 — New environment required

Correct high-level sequence:

```text
Terraform
  -> AWS infrastructure
      |
      v
Ansible
  -> roles / operational setup
      |
      v
Liquibase
  -> schema
      |
      v
Application
```

Jenkins can orchestrate the sequence.

---

# Incident 15 — Production schema drift

Symptoms:

```text
Production has an object not present in Git
```

Possible causes:

- emergency SQL
- manual DBA change
- old deployment process
- failed migration
- wrong environment

Response:

```text
Compare
 -> identify owner
 -> decide desired state
 -> create corrective changeset
 -> deploy through normal process
```

Do not simply delete the unexpected object because Git does not contain it.

---

# Master incident matrix

| Incident | Terraform | Ansible | Liquibase | Jenkins |
|---|---:|---:|---:|---:|
| RDS storage expansion | Primary | | | Orchestrate |
| Security group correction | Primary | | | Orchestrate |
| Missing DB role | | Primary | | Orchestrate |
| Grant correction | | Primary | | Orchestrate |
| Create application table | | | Primary | Orchestrate |
| Add production index | | | Primary | Orchestrate |
| Schema release | | | Primary | Primary orchestration |
| Session investigation | | Useful | | Optional |
| Emergency SQL | | Optional | Later reconcile | Optional |
| Existing RDS import | Primary | | | Orchestrate |
| Daily health report | | Primary | | Schedule |
| Full release | Infrastructure | Operations | Schema | Primary orchestration |
