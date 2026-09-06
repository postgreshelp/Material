# 05 - Liquibase Project

## Structure

```text
liquibase/
├── changelog/
│   ├── 001-create-schema.sql
│   └── 002-create-table.sql
├── db.changelog-master.yaml
└── liquibase.properties
```

## Configure properties

Edit:

```bash
vi liquibase/liquibase.properties
```

Use:

```properties
url: jdbc:postgresql://<RDS_ENDPOINT>:5432/postgres
username: postgres
password: <RDS_PASSWORD>
changelog-file: db.changelog-master.yaml
```

## Validate

```bash
cd ~/ansible-postgresql/liquibase
liquibase validate
```

Expected:

```text
No validation errors found.
Liquibase command 'validate' was executed successfully.
```

## Check status

```bash
liquibase status
```

The first run should show the two changesets as not applied.

## Apply

```bash
liquibase update
```

## Verify

```bash
psql -h <RDS_ENDPOINT> -U postgres -d postgres
```

Then:

```sql
\dn
\dt paylite.*
```

Expected:

```text
paylite.employee
```

Liquibase also maintains `databasechangelog` and `databasechangeloglock`.
