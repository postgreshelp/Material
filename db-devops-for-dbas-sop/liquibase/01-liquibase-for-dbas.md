# Liquibase for PostgreSQL DBAs

## 1. What is Liquibase?

Liquibase is a database change-management tool.

DBA mental model:

> **"How do I make database schema changes versioned, reviewable, repeatable and deployable?"**

Liquibase is the database equivalent of treating schema changes as source-controlled release artifacts.

---

# 2. Nomenclature

## Changelog

The collection of database changes Liquibase manages.

## Changeset

One uniquely identified database change.

Typical identity:

```text
id + author + filepath
```

Example:

```sql
--changeset paylite:001
```

## Formatted SQL

SQL files containing Liquibase metadata comments.

Example:

```sql
--liquibase formatted sql
--changeset paylite:001
```

## Master changelog

The top-level changelog that includes other changelogs.

## DATABASECHANGELOG

Liquibase's tracking table containing executed changeset information.

## DATABASECHANGELOGLOCK

Used to coordinate Liquibase execution so concurrent updates do not corrupt deployment state.

## Tag

A named point in database release history.

## Rollback

Reverses deployed changes. With formatted SQL, rollback logic needs to be explicitly written; modeled YAML/XML/JSON change types may support generated rollback for supported operations. citeturn0search9turn0search14

---

# 3. Important commands

## Version

```bash
liquibase --version
```

## Validate

```bash
liquibase validate
```

Use before deployment.

## Status

```bash
liquibase status
```

Shows pending changesets.

## Update

```bash
liquibase update
```

Deploys pending changesets.

## History

```bash
liquibase history
```

Useful for deployment history when supported by the installed version/edition.

## Rollback

Use only when rollback design and target are explicitly understood.

Examples vary by release/edition and rollback strategy.

---

# 4. Production DBA lifecycle

```text
Developer creates SQL
        |
        v
Liquibase changeset
        |
        v
Git review
        |
        v
validate
        |
        v
test database
        |
        v
Jenkins approval
        |
        v
Liquibase update
        |
        v
Production PostgreSQL
        |
        v
DATABASECHANGELOG
```

---

# 5. Real production incidents

## Incident: Application release requires a new column

Requirement:

```text
Add employee.email
```

Do not run ad-hoc production SQL if the organization's deployment model uses Liquibase.

Create:

```text
003-add-email.sql
```

Example:

```sql
--liquibase formatted sql

--changeset paylite:003

ALTER TABLE paylite.employee
ADD COLUMN email VARCHAR(255);
```

Then:

```bash
liquibase validate
liquibase status
liquibase update
```

The changeset becomes part of deployment history.

---

## Incident: Index required for production performance

Symptom:

```text
A query is repeatedly scanning a large table.
```

After SQL analysis confirms an index is appropriate:

```text
Liquibase changeset
        |
        v
CREATE INDEX
        |
        v
test
        |
        v
production
```

This is preferable to an undocumented manual index command when schema is controlled by Liquibase.

---

## Incident: Release deployment partially failed

DBA sees a failed changeset.

First inspect:

```bash
liquibase status
```

Then inspect:

```sql
SELECT *
FROM databasechangelog
ORDER BY dateexecuted;
```

Do not blindly rerun destructive SQL.

Determine:

1. Which changeset executed?
2. Which changeset failed?
3. Was the database partially modified?
4. Is the changeset safely rerunnable?
5. Is a corrective changeset preferable?
6. Is rollback actually supported and tested?

---

## Incident: Bad schema release

If rollback is required, use the organization's approved Liquibase rollback procedure.

Important:

> Formatted SQL changelogs require explicit rollback logic; do not assume Liquibase can automatically reverse arbitrary SQL. citeturn0search14

---

## Incident: Two Jenkins jobs deploy simultaneously

Liquibase's database changelog lock mechanism helps coordinate Liquibase update operations.

The DBA should still investigate why two production deployments were allowed to start simultaneously and fix the CI/CD control.

---

# 6. Liquibase vs Ansible

| Requirement | Tool |
|---|---|
| Create application role | Ansible |
| Grant CONNECT | Ansible |
| Create table as release artifact | Liquibase |
| Add index as application release | Liquibase |
| Check sessions | Ansible/SQL |
| Run schema migration | Liquibase |
| Execute the complete workflow | Jenkins |

---

# 7. Liquibase production rules

## Rule 1

Never edit an already deployed changeset to change its meaning.

Create a new changeset.

## Rule 2

Numbering is human-friendly, but the changeset identity is what matters.

## Rule 3

Test migrations against a production-like database.

## Rule 4

Large table changes require special planning.

Examples:

- large `ALTER TABLE`
- index creation
- type conversion
- data backfill
- NOT NULL enforcement

## Rule 5

Rollback is not automatically guaranteed.

Design rollback or forward-fix strategy before production.

---

# 8. Schema refresh / drift verification

A DBA can use Liquibase to compare expected database state with an environment, depending on the selected Liquibase workflow and edition.

Do not confuse:

```text
database refresh
```

with:

```text
restoring production data into another environment
```

Those are different operations.

---

# 9. Golden rule

```text
Liquibase = database schema release management
```
