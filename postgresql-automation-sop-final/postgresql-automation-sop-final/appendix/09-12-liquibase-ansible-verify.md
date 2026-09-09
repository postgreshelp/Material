# 04 · Lab — Liquibase, Ansible, Prerequisites & Verify (09–12)

**PostgresHelp Labs**

Liquibase, Ansible, and the Java/PostgreSQL prerequisites they depend on — installed on the Linux host that acts as the Ansible control node (`dnf`-based, Amazon Linux) — plus a final consolidated check across everything installed in steps 02–11.

---

## 09 · Liquibase Installation

Runs the changelogs in `liquibase/changelog/` against the Aurora cluster from topic 01. Installed on the Linux host, not the Windows workstation.

> **MENTAL MODEL**
> Liquibase is a Java program, not a native binary — it needs a JVM to run at all, and a JDBC driver to speak Postgres's wire protocol specifically. Neither is bundled by default: this topic is really three installs wearing one name.

> **BEFORE YOU START**
> - ✓ Java 21 installed (topic 11, or via topic 10's combined `dnf install`) — `java -version` works
> - ✓ Aurora cluster running and reachable (topic 01) — you have its endpoint hostname

**Steps**

1. Download the release tarball:
```bash
$ wget https://github.com/liquibase/liquibase/releases/download/v5.0.4/liquibase-5.0.4.tar.gz
```

2. Extract, locate, symlink.

> **WHY THIS COMMAND?** `find` before `ln -s` matters because release tarball layouts change between versions — guessing the launcher's path is how the symlink silently points at nothing.

```bash
$ mkdir -p /opt/liquibase
$ tar -xzf liquibase-5.0.4.tar.gz -C /opt/liquibase
$ find /opt/liquibase -maxdepth 2 -type f -name liquibase -o -name liquibase.sh
$ ln -s /opt/liquibase/liquibase /usr/local/bin/liquibase
```

3. Add the PostgreSQL JDBC driver — not bundled — Liquibase needs this jar in its `lib` folder to talk to Aurora PostgreSQL:
```bash
$ curl -L -o /opt/liquibase/lib/postgresql.jar https://jdbc.postgresql.org/download/postgresql-42.7.8.jar
```

4. Point it at the cluster — `liquibase/liquibase.properties`, copy as-is:
```properties
url: jdbc:postgresql://mypgrds.cijxwe4ckz1m.us-east-1.rds.amazonaws.com:5432/postgres
username: postgres
password: postgres
changelog-file: db.changelog-master.yaml
```

> **WHY THIS COMMAND?** `validate` and `status` before `update` exist for the same reason `terraform plan` exists before `apply`: see what's about to change before it's irreversible.

```bash
$ cd liquibase
$ liquibase validate
$ liquibase status
$ liquibase update
```

> **EXPECTED RESULT**
> - **validate:** "No validation errors found" — changelog syntax is well-formed; nothing touched the DB yet.
> - **update:** "UPDATE SUMMARY / Run: 2" and "Liquibase: Update has been successful."
> - **If it fails:** A JDBC/driver error means step 3 (the driver jar) is missing or in the wrong folder — not a credentials problem.

**What changed?**
```
Before                                  After
paylite database                        paylite database
└── (schema paylite doesn't exist)       ├── schema  paylite
                                         ├── table  paylite.employee
                                         └── DATABASECHANGELOG  (2 rows: paylite:001, paylite:002)
```
> Two changesets applied, in order, and both are now permanently recorded in the ledger — a second `liquibase update` right now would do nothing.

```bash
$ liquibase --version
Liquibase Version: 5.0.4

$ liquibase update
Running Changeset: changelog/001-create-schema.sql::paylite:001::paylite
Running Changeset: changelog/002-create-table.sql::paylite:002::paylite

UPDATE SUMMARY
Run:                          2
Previously run:               0
Filtered out:                 0
-------------------------------
Total change sets:            2

Liquibase: Update has been successful.
```

---

## 10 · Ansible Installation

Installed straight from distro repos — this box is both the Ansible control node and the target for `ansible_connection=local` in `inventory/hosts`.

> **MENTAL MODEL**
> Ansible core (the engine) and its Postgres-specific modules (the collection) ship separately on purpose — most Ansible installs never touch a database, so bundling every domain's modules by default would be dead weight for everyone else.

> **BEFORE YOU START**
> - ✓ Root or sudo access on the Linux host
> - ✓ `dnf` available (Amazon Linux / RHEL-family)

**Steps**

1. Install ansible-core (bundled with Java in one shot).

> **WHY THIS COMMAND?** Both packages come from `dnf`, so installing them together in one line means one less thing to remember later, and satisfies topic 11's Java requirement at the same time.

```bash
$ sudo dnf install -y java-21-amazon-corretto ansible-core
```

2. Add the PostgreSQL collection — the playbooks in `playbooks/` use `community.postgresql.*` modules:
```bash
$ ansible-galaxy collection install community.postgresql
```

> **EXPECTED RESULT**
> - **Look for:** `ansible [core 2.17.x]` and a Python version line.
> - **--syntax-check:** prints just the playbook path — no task output, because nothing actually ran.

```bash
$ ansible --version
ansible [core 2.17.x]
  python version = 3.x

$ ansible-galaxy collection list | grep postgresql
community.postgresql   3.x.x

$ ansible-playbook playbooks/postgresql_admin.yml --syntax-check
playbook: playbooks/postgresql_admin.yml
```

---

## 11 · Java / PostgreSQL Prerequisites

The two runtime dependencies that Liquibase (09) and manual DB checks rely on — both via `dnf`.

> **MENTAL MODEL**
> This topic is listed after 09 and 10 in the course numbering, but Java is actually a hard dependency *of* topic 09 — it was already satisfied back in topic 10's combined install line. Only the PostgreSQL client is genuinely new here.

> **Already done** — Java 21 was installed in topic 10's single `dnf install` line alongside `ansible-core`. Only the PostgreSQL package is new below.

**Java (reference — from topic 10)**
```bash
$ sudo dnf install -y java-21-amazon-corretto ansible-core
```

**PostgreSQL 18 (new)**
```bash
$ sudo dnf install -y postgresql18-server
```

> **EXPECTED RESULT**
> - **java -version:** "openjdk version 21.0.x" and "Corretto-21.0.x" — confirms it's actually the Corretto build, not some other JDK already on PATH.
> - **psql connects?** Reaching a `postgres=>` prompt confirms both the client install AND that topic 01's security group actually allows this connection.

```bash
$ java -version
openjdk version "21.0.x" 2024-xx-xx LTS
OpenJDK Runtime Environment Corretto-21.0.x.x.1 (build 21.0.x+9-LTS)

$ psql --version
psql (PostgreSQL) 18.x

$ psql -h mypgrds.cijxwe4ckz1m.us-east-1.rds.amazonaws.com -U postgres -d postgres
postgres=>
```

---

## 12 · Verify Everything

One pass across every tool installed in 02–11, plus an end-to-end smoke test of the actual project.

> **MENTAL MODEL**
> Each row below answers one question in isolation. The smoke test after it answers a different question: do the tools actually agree with each other about the state of the system, not just "am I installed."

| Tool | Command | Expect |
|---|---|---|
| VS Code | `code --version` | version string, no error |
| Git | `git --version` | `git version 2.4x...` |
| GitHub auth | `ssh -T git@github.com` | "successfully authenticated" |
| Copilot | type a comment in any file | inline grey suggestion appears |
| AWS CLI | `aws --version` | `aws-cli/2.x` |
| AWS auth | `aws sts get-caller-identity` | returns Account/Arn JSON |
| Terraform | `terraform -version` | `Terraform v1.9.x` |
| Liquibase | `liquibase --version` | `Liquibase Version: 5.0.4` |
| Ansible | `ansible --version` | `ansible [core 2.17.x]` |
| Java | `java -version` | `openjdk version "21...` |
| psql | `psql --version` | `psql (PostgreSQL) 18.x` |

> **WHY THIS COMMAND?** Each tool passing its own version check doesn't prove the pipeline works end to end — `terraform validate` + `liquibase validate` + a real playbook run is the only way to confirm they agree on what actually exists.

**End-to-end smoke test**
```powershell
PS C:\Users\hp\NewTerraform> terraform validate
Success! The configuration is valid.
```
```bash
$ liquibase validate
No validation errors found

$ ansible-playbook playbooks/postgresql_admin.yml
TASK [Display PostgreSQL version] ***
ok: [localhost] => {
    "postgres_version.query_result": [
        { "version": "PostgreSQL 16.4 on x86_64-pc-linux-gnu, ... Aurora PostgreSQL" }
    ]
}

PLAY RECAP ***
localhost : ok=4 changed=0 unreachable=0 failed=0
```

> **EXPECTED RESULT**
> - **terraform validate:** "Success!" — syntax and internal consistency only, doesn't confirm AWS matches.
> - **ansible ok=4:** Four tasks ran with zero `changed` and zero `failed` — a healthy idempotent re-run, not a first run.

---

## Can I explain this? (topics 09–12)

- [ ] Why does Liquibase need a separate JDBC driver jar instead of shipping one?
- [ ] Why is Java actually a topic-09 dependency, not just a topic-11 one?
- [ ] What's the difference between `ansible-core` and a collection like `community.postgresql`?
- [ ] In the smoke test, what does `ok=4 changed=0` tell you that a first run wouldn't show?
- [ ] Without looking — list every "Look for" phrase from this page's Expected Result boxes, from memory.

---

Steps 09–11 run on the Linux host (Ansible control node); 02–08 run on the Windows workstation — worth keeping that split explicit when reproducing this.

*PostgresHelp Labs · Build. Operate. Automate. Grow.*
