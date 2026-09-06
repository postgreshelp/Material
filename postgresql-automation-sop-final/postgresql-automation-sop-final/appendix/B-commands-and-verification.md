# Appendix B - Commands and Verification

## Ansible

```bash
ansible --version
ansible-inventory --graph
ansible postgresql -m ping
ansible-galaxy collection list | grep postgresql
```

## PostgreSQL

```bash
psql --version
psql -h <RDS_ENDPOINT> -U postgres -d postgres
```

```sql
SELECT version();
\du
\dn
\dt paylite.*
```

## Liquibase tracking tables

```sql
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_name IN ('databasechangelog', 'databasechangeloglock')
ORDER BY table_schema, table_name;
```

## Liquibase

```bash
cd ~/ansible-postgresql/liquibase
liquibase --version
liquibase validate
liquibase status
liquibase update
```

## Jenkins

```bash
sudo systemctl status jenkins
sudo -u jenkins java -version
sudo -u jenkins ansible-galaxy collection list | grep postgresql
sudo journalctl -u jenkins --no-pager -n 50
