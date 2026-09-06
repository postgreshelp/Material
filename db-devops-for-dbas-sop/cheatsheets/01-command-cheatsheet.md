# DBA DevOps Command Cheatsheet

## Terraform

```bash
terraform fmt
terraform fmt -check
terraform init
terraform validate
terraform plan
terraform plan -out=tfplan
terraform apply
terraform apply tfplan
terraform show
terraform state list
terraform state show <ADDRESS>
terraform output
terraform providers
terraform import <ADDRESS> <ID>
terraform destroy
```

### Most important production command

```bash
terraform plan
```

Review the plan before `apply`.

---

## Ansible

```bash
ansible --version
ansible-inventory --graph
ansible-inventory --list
ansible postgresql -m ping
ansible-galaxy collection list
ansible-galaxy collection install community.postgresql
ansible-playbook playbooks/postgresql_role.yml
ansible-playbook playbooks/postgresql_admin.yml
ansible-playbook playbooks/postgresql_admin_full.yml
ansible-playbook playbooks/liquibase_deploy.yml
ansible-playbook playbooks/liquibase_deploy.yml --check
ansible-playbook playbooks/liquibase_deploy.yml -vv
ansible-playbook site.yml --limit postgresql
ansible-playbook site.yml --syntax-check
```

---

## PostgreSQL

```bash
psql -h <HOST> -U <USER> -d <DATABASE>
```

Useful:

```sql
SELECT version();

SELECT count(*) FROM pg_stat_activity;

SELECT pid, usename, datname, state, query
FROM pg_stat_activity
ORDER BY pid;

SELECT pg_size_pretty(pg_database_size(current_database()));

SELECT *
FROM pg_stat_replication;

\du
\dn
\dt *.*
```

---

## Liquibase

```bash
liquibase --version
liquibase validate
liquibase status
liquibase update
liquibase history
```

Tracking:

```sql
SELECT id, author, filepath, dateexecuted
FROM databasechangelog
ORDER BY dateexecuted;
```

---

## Jenkins service on Linux

```bash
sudo systemctl status jenkins
sudo systemctl restart jenkins
sudo journalctl -u jenkins --no-pager
sudo journalctl -u jenkins --no-pager -n 100
sudo -u jenkins java -version
sudo -u jenkins ansible-galaxy collection list
```

---

# One-line decision cheat sheet

```text
AWS infrastructure?        -> Terraform
Repeated DBA operation?    -> Ansible
Schema/database release?   -> Liquibase
Automate/approve/schedule? -> Jenkins
Immediate SQL investigation? -> PostgreSQL tools
```

# Production pre-change checklist

```text
[ ] Correct environment
[ ] Correct database
[ ] Correct Git commit
[ ] Correct credentials
[ ] Terraform plan reviewed
[ ] Ansible target reviewed
[ ] Liquibase status reviewed
[ ] Backup/recovery strategy understood
[ ] Lock/concurrency checked
[ ] Monitoring available
[ ] Rollback or forward-fix plan understood
[ ] Post-change verification defined
```
