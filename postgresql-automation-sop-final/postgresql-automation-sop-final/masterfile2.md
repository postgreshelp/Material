
### Install software

```
sudodnf install -y java-21-amazon-corretto ansible-core
ansible --version

wget https://github.com/liquibase/liquibase/releases/download/v5.0.4/liquibase-5.0.4.tar.gz
mkdir -p /opt/liquibase
tar -xzf liquibase-5.0.4.tar.gz -C /opt/liquibase
find /opt/liquibase -maxdepth 2 -type f -name liquibase -o -name liquibase.sh
ln -s /opt/liquibase/liquibase /usr/local/bin/liquibase
liquibase --version
```

### Files 
```
[root@ip-10-10-1-234 ansible-postgresql]# find . -maxdepth 3 -type f
./ansible.cfg
./inventory/hosts
./inventory/group_vars/postgresql.yml
./touch
./playbook/postgresql_role.yml
./playbooks/postgresql_role.yml
./playbooks/postgresql_admin.yml
./playbooks/postgresql_admin_full.yml
./playbooks/liquibase_deploy.yml
./liquibase/changelog/001-create-schema.sql
./liquibase/changelog/002-create-table.sql
./liquibase/db.changelog-master.yaml
./liquibase/liquibase.properties
```






curl -L -o lib/postgresql.jar https://jdbc.postgresql.org/download/postgresql-42.7.8.jar

ls -lh lib/postgresql.jar
