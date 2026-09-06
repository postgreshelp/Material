# Appendix A - Troubleshooting

This section preserves setup problems encountered during the lab without polluting the main SOP.

## Jenkins uses Java 17

Symptom:

```text
Running with Java 17 ... older than the minimum required version (Java 21).
Supported Java versions are: [21, 25]
```

Check:

```bash
java -version
sudo -u jenkins java -version
sudo journalctl -u jenkins --no-pager -n 50
```

Ensure the Jenkins service uses Java 21, then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

## Missing psycopg2

```bash
sudo dnf install python3-psycopg2 -y
```

Verify:

```bash
python3 -c "import psycopg2; print(psycopg2.__version__)"
```

## Missing Liquibase PostgreSQL driver

Symptom:

```text
Cannot find database driver: org.postgresql.Driver
```

Install the JDBC jar:

```bash
cd /opt/liquibase
wget https://jdbc.postgresql.org/download/postgresql-42.7.8.jar
cp postgresql-42.7.8.jar /opt/liquibase/lib/
```

## Liquibase cannot find changelog

Run from the project directory:

```bash
cd ~/ansible-postgresql/liquibase
liquibase status
```

## Jenkins cannot see community.postgresql

Install for the Jenkins account:

```bash
sudo -u jenkins ansible-galaxy collection install community.postgresql
```

## Disk-space/node issue

Check:

```bash
df -h
df -h /tmp
```

Clean unnecessary files and rerun Jenkins if the node is marked offline because of disk-space thresholds.
