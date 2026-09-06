# 04 - Java 21 and Liquibase

## Install Java 21

Install Java 21 before Jenkins setup:

```bash
sudo dnf install java-21-amazon-corretto -y
java -version
```

If needed:

```bash
sudo alternatives --config java
java -version
```

## Install Liquibase 5.0.4

Install Liquibase under:

```text
/opt/liquibase
```

Verify:

```bash
liquibase --version
```

Expected:

```text
Liquibase Version: 5.0.4
```

## Install PostgreSQL JDBC driver

```bash
cd /opt/liquibase
wget https://jdbc.postgresql.org/download/postgresql-42.7.8.jar
cp postgresql-42.7.8.jar /opt/liquibase/lib/
```

Verify:

```bash
liquibase --version
```

The PostgreSQL JDBC driver should appear in the listed libraries.
