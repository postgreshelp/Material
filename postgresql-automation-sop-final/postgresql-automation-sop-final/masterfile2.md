
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

dnf install postgresql18-server
```

### pull the files

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

### commands

```
[root@ip-10-10-1-234 ansible-postgresql]# ansible-inventory --graph
@all:
  |--@ungrouped:
  |--@postgresql:
  |  |--localhost
[root@ip-10-10-1-234 ansible-postgresql]# ansible postgresql -m ping
[WARNING]: Platform linux on host localhost is using the discovered Python interpreter at /usr/bin/python3.9, but future
installation of another Python interpreter could change the meaning of that path. See https://docs.ansible.com/ansible-
core/2.15/reference_appendices/interpreter_discovery.html for more information.
localhost | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.9"
    },
    "changed": false,
    "ping": "pong"
}
```

### install collection

```
ansible-galaxy collection install community.postgresql
ansible-galaxy collection list | grep postgresql
```
```
### Run playbooks
ansible-playbook playbooks/postgresql_role.yml

### pre-requisite missing
dnf install python3-psycopg2 -y

### Verify
psql -h mypgrds.cijxwe4ckz1m.us-east-1.rds.amazonaws.com -U postgres -d postgres

### couple of more playbooks
ansible-playbook playbooks/postgresql_admin.yml
ansible-playbook playbooks/postgresql_admin_full.yml

### Liquibase

curl -L -o lib/postgresql.jar https://jdbc.postgresql.org/download/postgresql-42.7.8.jar
ls -lh lib/postgresql.jar

 cd ~/ansible-postgresql/liquibase
liquibase validate
liquibase update
liquibase status

### Verify
psql -h mypgrds.cijxwe4ckz1m.us-east-1.rds.amazonaws.com -U postgres -d postgres

### Bring in ansible + Liquibase
 cd ~/ansible-postgresq
ansible-playbook playbooks/liquibase_deploy.yml

### Jenkins
 sudo wget -O /etc/yum.repos.d/jenkins.repo \
  https://pkg.jenkins.io/redhat-stable/jenkins.repo
  
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2026.key
sudo dnf install jenkins -y
sudo systemctl enable --now jenkins

sudo systemctl start jenkins
sudo systemctl status jenkins

sudo -u jenkins ansible-galaxy collection list | grep community.postgresql
sudo -u jenkins ansible-galaxy collection install community.postgresql

 sudo -u jenkins env ANSIBLE_CONFIG=/opt/ansible-postgresql/ansible.cfg \
ansible-playbook /opt/ansible-postgresql/playbooks/liquibase_deploy.yml
```

curl -L -o lib/postgresql.jar https://jdbc.postgresql.org/download/postgresql-42.7.8.jar

ls -lh lib/postgresql.jar
