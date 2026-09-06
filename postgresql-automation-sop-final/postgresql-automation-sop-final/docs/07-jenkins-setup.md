# 07 - Jenkins Setup

## Java 21 prerequisite

Verify:

```bash
java -version
```

After Jenkins is installed:

```bash
sudo -u jenkins java -version
```

Jenkins must use Java 21.

## Install Jenkins

Install the Jenkins package using the official Jenkins repository instructions for Amazon Linux 2023.

Enable and start:

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Verify:

```bash
sudo systemctl status jenkins
```

Logs:

```bash
sudo journalctl -u jenkins --no-pager -n 50
```

## Install the PostgreSQL Ansible collection for Jenkins

The Jenkins user has its own Ansible collection path:

```bash
sudo -u jenkins ansible-galaxy collection install community.postgresql
```

Verify:

```bash
sudo -u jenkins ansible-galaxy collection list | grep postgresql
```

## Jenkins repository

Jenkins must have the repository containing:

```text
ansible.cfg
inventory/
playbooks/
liquibase/
jenkins/Jenkinsfile
```

Prefer source control checkout rather than copying files manually.

## Credentials

Do not commit the real RDS password. Use Jenkins Credentials for an actual CI/CD environment.
