# 05 · Reference — Command Cheatsheet

**PostgresHelp Labs**

> **Reference only.** No concepts explained here — for the why behind any command, see [01 Theory](13-theory-manual-toolchain.md), [02 Terraform](01-terraform-vpc-aurora-reference.md), or the 03/04 Lab pages.

Every command run across topics 01–12, with what each flag actually does.

---

## Git

| Command | Flag | Meaning |
|---|---|---|
| `git init` | — | Turns the current folder into a Git repository (creates `.git/`). |
| `git config user.name "..."` | `--global` | Sets the author name for commits. `--global` applies it to every repo on this machine, not just this one. |
| `git config user.email "..."` | `--global` | Sets the author email for commits — shows up in `git log` and GitHub. |
| `git config --list` | — | Prints all active config values, so you can confirm name/email were actually set. |
| `git remote add origin <url>` | — | Links this local repo to a remote (GitHub) named `origin` — the default name, not a keyword. |
| `git add .` | `.` | Stages every changed/new file in the current folder for the next commit. `.` means "here and below." |
| `git status` | — | Shows what's staged, what's changed but unstaged, and what's untracked. |
| `git commit` | `-m "msg"` | `-m` = message. Passes the commit message inline, instead of opening your default editor. |
| `git branch` | `-M master` | `-M` = move/rename (force). Renames the current branch to `master`, overwriting if that name exists. |
| `git push` | `-u origin master` | `-u` = set upstream. Pushes and remembers the pairing so future `push`/`pull` need no arguments. |
| `git pull` | — | Fetches remote commits and merges them into your current branch, in one step. |
| `ssh-keygen` | `-t ed25519 -C "email"` | `-t` = key type/algorithm. `-C` = comment, usually your email, to label the key on GitHub. |
| `ssh git@github.com` | `-T` | `-T` = disable pseudo-terminal. Tests SSH auth against GitHub without opening a shell. |

## Terraform

| Command | Flag | Meaning |
|---|---|---|
| `terraform init` | — | Downloads the providers your config needs and sets up the local backend. Run once per new/changed config. |
| `terraform validate` | — | Checks your `.tf` files are syntactically correct and internally consistent. Fully offline. |
| `terraform fmt` | — | Auto-rewrites your files into Terraform's canonical formatting. |
| `terraform plan` | — | Shows exactly what would change if you applied — a dry run. Always read this before `apply`. |
| `terraform apply` | — | Actually creates/updates real AWS resources to match your config. Prompts for a typed `yes`. |
| `terraform import` | `<addr> <id>` | Maps a real AWS resource onto a resource block already in your config — writes state only. |
| `terraform -version` | — | Prints the installed Terraform version and resolved provider versions. |

## AWS CLI

| Command | Flag | Meaning |
|---|---|---|
| `aws --version` | — | Confirms the CLI is installed and on PATH. |
| `aws configure` | — | Interactively writes your access key, secret key, default region, and output format to `~/.aws/`. |
| `aws sts get-caller-identity` | — | Returns the IAM identity your current credentials resolve to. |
| `aws ec2 describe-vpcs` | `--vpc-ids <id>` | Looks up details for one specific VPC by ID. |

## Ansible

| Command | Flag | Meaning |
|---|---|---|
| `ansible --version` | — | Shows the installed Ansible core version and the Python interpreter. |
| `ansible-galaxy collection install` | `community.postgresql` | Downloads a collection — extra modules not in Ansible core. |
| `ansible-galaxy collection list` | — | Lists every collection currently installed. |
| `ansible-playbook <file>.yml` | `--syntax-check` | Parses the playbook's syntax without connecting to any host. |
| `ansible-playbook <file>.yml` | — | Actually runs the playbook against the hosts in your inventory. |

## Liquibase

| Command | Flag | Meaning |
|---|---|---|
| `liquibase --version` | — | Confirms Liquibase is installed and on PATH. |
| `liquibase validate` | — | Checks the changelog file(s) are well-formed — no DB connection changes made. |
| `liquibase status` | — | Connects to the target DB and lists which changesets haven't been applied yet. |
| `liquibase update` | — | Applies every pending changeset, in order, and records each in `DATABASECHANGELOG`. |

## PostgreSQL / psql

| Command | Flag | Meaning |
|---|---|---|
| `psql --version` | — | Confirms the PostgreSQL client is installed. |
| `psql` | `-h <host> -U <user> -d <db>` | `-h` host, `-U` user, `-d` database. |
| `java -version` | — | Confirms the JDK is installed — a hard dependency for Liquibase. |

## System / Linux (install & file handling)

| Command | Flag | Meaning |
|---|---|---|
| `sudo dnf install -y <pkg>` | `-y` | Auto-confirm install via the Amazon Linux/RHEL/Fedora package manager. |
| `wget <url>` | — | Downloads a file from a URL into the current directory. |
| `tar` | `-xzf <file> -C <dir>` | `-x` extract, `-z` gzip, `-f` file, `-C` target directory. |
| `ln -s <target> <link>` | `-s` | Creates a symbolic shortcut pointing at the target. |
| `curl` | `-L -o <file> <url>` | `-L` follows redirects, `-o` saves to a named file. |
| `find <dir>` | `-maxdepth 2 -name <pat>` | Limits search depth; matches a filename pattern. |
| `mkdir` | `-p <dir>` | Creates the folder (and missing parents) without erroring if it exists. |
| `code` | `--install-extension <id>` | Installs a VS Code extension from the command line. |

## PowerShell — audit what's installed / configured (Windows)

| Command | Flag | Meaning |
|---|---|---|
| `Get-Command terraform` | — | Confirms `terraform.exe` is on PATH. Errors if not installed. |
| `Get-Command git` | — | Same PATH check, for Git. |
| `git config user.name` | `--global` | Prints just the configured commit author name. |
| `git config user.email` | `--global` | Prints just the configured commit author email. |
| `git config --list` | `--global --show-origin` | `--show-origin` adds which config file each setting came from. |
| `Get-Command aws` | — | Confirms the AWS CLI is on PATH. |
| `aws configure list` | — | Shows which profile is active and where each credential value is read from. |
| `Get-Command psql` | `-ErrorAction SilentlyContinue` | Checks if `psql` is on PATH without printing an error if missing. |
| `Get-Service -Name postgresql*` | — | Lists any locally installed PostgreSQL Windows service. Empty = no local server. |
| `Get-ItemProperty` | `HKLM:\...\Uninstall\* \| where DisplayName -like "*PostgreSQL*"` | Registry check — catches an MSI-installed PostgreSQL even with a stopped service. |
| `Test-NetConnection` | `-ComputerName localhost -Port 5432` | Checks if anything is listening on Postgres's default port. |
| `$env:Path -split ';'` | — | Prints every folder on PATH — pipe to `Select-String terraform` to isolate one tool. |
| `whoami` | — | The current Windows user — distinct from the Git author or AWS IAM identity. |

---

## Appendix — other commands worth learning

Not used in this project yet, but commonly needed the moment things go wrong or the project grows.

**Git**
- `git log --oneline` — compact commit history
- `git diff` — see unstaged changes
- `git stash` — shelve changes temporarily
- `git rebase -i` — rewrite/squash commit history
- `git revert <hash>` — undo a commit safely

**Terraform**
- `terraform destroy` — tear down everything managed
- `terraform state list` — list tracked resources
- `terraform state show <addr>` — inspect one resource's state
- `terraform taint <addr>` — force recreation on next apply
- `terraform workspace new <name>` — isolate state per environment

**Ansible**
- `ansible all -m ping` — ad-hoc connectivity check
- `ansible-playbook --check` — dry run, no changes made
- `ansible-vault encrypt <file>` — encrypt secrets at rest
- `ansible-inventory --list` — dump the resolved inventory

**Liquibase**
- `liquibase rollback <tag>` — revert to a tagged point
- `liquibase rollbackCount <n>` — undo the last N changesets
- `liquibase diff` — compare two databases' schemas
- `liquibase generateChangeLog` — reverse-engineer a changelog

**AWS CLI**
- `aws s3 ls` — list buckets/objects
- `aws iam list-users` — list IAM users
- `aws rds describe-db-clusters` — inspect Aurora clusters
- `aws logs tail <log-group> --follow` — stream CloudWatch logs

**psql & Linux ops**
- `\l` \ `\dt` \ `\du` — list DBs / tables / roles
- `EXPLAIN ANALYZE <query>` — see the real query plan
- `pg_dump` / `pg_restore` — backup and restore
- `systemctl status <service>` — check a service's state
- `journalctl -u <service> -f` — tail a service's logs

**PowerShell / Windows**
- `winget list` — every winget-installed package + version
- `Get-Package` — broader installed-software inventory
- `Get-NetTCPConnection -LocalPort 5432` — what's bound to that port
- `Get-Service | Where-Object Status -eq 'Running'` — all running services
- `$env:JAVA_HOME` / `$env:AWS_PROFILE` — check an env var's value

---
*PostgresHelp Labs · Build. Operate. Automate. Grow.*
