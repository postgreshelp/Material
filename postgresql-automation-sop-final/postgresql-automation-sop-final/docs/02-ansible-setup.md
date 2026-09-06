# 02 - Ansible Setup

## Install Ansible

```bash
sudo dnf install ansible-core -y
ansible --version
```

## Install PostgreSQL client

Install the PostgreSQL client for the lab environment:

```bash
psql --version
```

Expected lab result:

```text
psql (PostgreSQL) 18.6
```

## Install psycopg2

```bash
sudo dnf install python3-psycopg2 -y
python3 -c "import psycopg2; print(psycopg2.__version__)"
```

## Install the PostgreSQL collection

```bash
ansible-galaxy collection install community.postgresql
ansible-galaxy collection list | grep postgresql
```

Expected lab collection:

```text
community.postgresql 4.2.0
```

## Configure the project

```bash
mkdir -p ~/ansible-postgresql
cd ~/ansible-postgresql
```

Copy this repository into the directory.

Verify:

```bash
ansible-inventory --graph
ansible postgresql -m ping
```
