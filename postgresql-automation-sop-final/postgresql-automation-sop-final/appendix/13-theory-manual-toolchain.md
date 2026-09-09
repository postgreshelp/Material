# 01 · Theory — The Toolchain

**PostgresHelp Labs**

Six tools, one pipeline. Before any lab, this is the mental model — what job each tool does, how it's architected, and where it deliberately stops and hands off to the next tool.

> **CORE IDEA**
> 1. Terraform provisions infrastructure.
> 2. Ansible configures what's running on that infrastructure.
> 3. Liquibase version-controls the database schema.
> 4. Jenkins runs all three, in order, triggered by Git.
> 5. Grafana watches the result — it changes nothing.
> 6. Git is the source of truth everything else reacts to.

## The full pipeline, end to end

```
Developer commits ──▶ Git  (source of truth: .tf, playbooks, changelogs, Jenkinsfile)
                       │
                       ▼
                 Jenkins  (orchestrates the pipeline, stage by stage)
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
  Terraform       Ansible       Liquibase
  provision       configure       version & deploy
  the VPC/RDS     the OS/DB       schema changes
        │              │              │
        └──────────────┼──────────────┘
                       ▼
                 running system
                       │
                       ▼
                 Grafana  (dashboards + alerts on what's actually happening)
```

Read it as: a commit is the only way in. Jenkins is the only thing that acts on a commit. Everything below Jenkins does exactly one job. Grafana only watches — it never writes back into the loop.

## Why split the work across six tools instead of one

Infra shape, machine state, schema state, orchestration, and visibility are not the same problem — a tool that tries to do all of them tends to do each one worse. Each page below covers one tool: what it is, its architecture, and real-world use cases.

| Page | Tool | Answers the question |
|---|---|---|
| 2 | Terraform | What infrastructure should exist? |
| 3 | Ansible | What state should a machine be in? |
| 4 | Liquibase | What should the database schema look like, in what order? |
| 5 | Jenkins & Grafana | Who runs all this, and how do we know it's healthy? |
| 6 | Git & GitOps | What triggers any of the above, and who's allowed to? |
| 7 | Knowledge check | Can I explain this without looking? |

---

## Page 2 · Provisioning — Terraform

Infrastructure as Code (IaC): infrastructure is described in text files, not clicked together in a console — so it can be reviewed, versioned, and reproduced exactly.

> **MENTAL MODEL**
> A shell script says *how* — run this command, then that one. Terraform's `.tf` files say *what* — "this VPC should exist with this CIDR." Terraform figures out the sequence of API calls needed to get from the current state to that description. Re-running it when nothing changed does nothing — a script would just run again.

```
.tf files (HCL)              ↳ your description of desired infrastructure
      │
      ▼
  Terraform Core  ──── reads / writes ────▶  terraform.tfstate
      │                                        ↳ Terraform's record of what it created last
      ▼
  Provider Plugin  (hashicorp/aws)
      │
      ▼
  AWS API  ──▶  VPC · Subnets · Aurora Cluster
```

Every `plan` diffs your .tf files against the state file, not against AWS directly — which is exactly why hand-editing a resource in the AWS console causes "drift" that Terraform will try to silently undo on the next apply.

| Concept | What it means |
|---|---|
| Provider | A plugin that knows how to talk to one platform's API (AWS, Azure, GitHub...). Translates HCL into real API calls. |
| Resource | One managed object — a VPC, a subnet, an RDS cluster. Terraform creates, updates, or destroys it to match your config. |
| Data source | A read-only lookup against something that already exists — never created or destroyed by Terraform. |
| State file | Terraform's map of resource-address → real-world ID. The only thing it trusts to know what it's already created. |
| Plan / Apply | Plan = dry-run diff. Apply = actually make the API calls. Splitting them is what makes Terraform safe to run in CI. |

> ⚠️ **Pitfall** — the state file often contains sensitive values (like the Aurora master password in this project) in plain text. In a real team, state is kept in a remote backend (S3 + a DynamoDB lock table is the common AWS pattern) with restricted access — never committed to Git.

**Real-world use cases**

- **Multi-cloud** — Companies running workloads across AWS, Azure, and GCP use one Terraform workflow for all three instead of learning each provider's own console — HashiCorp's own use-case guide leads with this.
- **Disaster recovery** — Because the environment is code, the same config can be re-applied in a different region. A region outage becomes a re-apply, not hours of manual console rebuilding.
- **Environment parity** — Dev, staging, and prod are the same `.tf` code with different variable values — so "it worked in staging" actually means something.

---

## Page 3 · Configuration Management — Ansible

Once infrastructure exists, something has to make sure the software *on* it is correct — right packages, right users, right replication config.

> **MENTAL MODEL**
> Ansible is agentless: it doesn't install anything permanent on the machines it manages. It connects over SSH, runs a set of tasks, and disconnects — leaving nothing behind to maintain or patch. Compare this to Puppet/Chef/SaltStack, which install a persistent agent that phones home on its own schedule.

```
  Control Node  (this machine, or the Linux host from topic 10)
      │  reads
      ▼
  inventory (hosts)  ── defines ──▶  which machines, grouped how
      │
      ▼
  Playbook (YAML)  ── calls ──▶  Modules / Collections
      │                                  ↳ e.g. community.postgresql.postgresql_query
      ▼
  SSH (Linux)  /  WinRM (Windows)
      │
      ▼
  Managed Node  ── no agent installed, no daemon left running
```

**Idempotency — the core promise.** Running the same playbook once, or fifty times, produces the same end state and no duplicate side effects. A task that says "this role should exist" checks first — it only creates the role if it's actually missing. This is what makes it safe to re-run a playbook after a partial failure.

| Concept | What it means |
|---|---|
| Inventory | The list of hosts (and groups) a playbook can target — static file or dynamic from a cloud API. |
| Playbook | A YAML file describing an ordered list of plays — each play maps tasks onto hosts. |
| Module | The actual unit of work (e.g. `postgresql_query`, `apt`, `copy`). |
| Collection | A packaged, versioned bundle of modules/roles for one domain — `community.postgresql` is one. |
| Handler | A task that only runs when notified — e.g. "restart postgresql" only if the config actually changed. |

> **Depends on Terraform** — Ansible needs a host to already exist and be reachable over SSH/WinRM. It's step two in the pipeline because it has nothing to configure until something has been provisioned.

**Real-world use cases**

- **Primary / replica** — One playbook configures both a PostgreSQL primary and its replicas consistently — instead of two servers drifting apart by hand.
- **Fleet-wide patching** — Red Hat lists this as a top enterprise use case: scheduled reboots and security-baseline enforcement across hundreds of servers.
- **Rolling deploys** — The `serial` keyword updates a fraction of a fleet at a time, so a bad rollout only affects a slice of hosts.

---

## Page 4 · Schema Version Control — Liquibase

Database schema changes are just as risky un-tracked as application code would be.

> **MENTAL MODEL**
> Liquibase treats every DDL change as a reviewable, ordered "changeset" instead of a script someone runs from memory. It keeps its own ledger inside the database of exactly which changesets already ran — so re-running `update` is always safe.

```
  db.changelog-master.yaml
      │  includes
      ▼
  001-create-schema.sql   (changeset paylite:001)
  002-create-table.sql    (changeset paylite:002)
      │
      ▼
  liquibase update
      │
      ▼
  Target Database
      ├── applies pending changesets, in order
      └── writes to  DATABASECHANGELOG
                       ↳ tracking table: which changesets ran, checksum, timestamp, author
```

On the next `update`, Liquibase reads `DATABASECHANGELOG` first — anything already listed there is skipped.

**Checksum protection.** Every applied changeset's checksum is stored alongside it. If someone edits a changeset *after* it already ran in production, the next `update` fails loudly with a checksum mismatch instead of silently re-running (or skipping) a modified script.

| Concept | What it means |
|---|---|
| Changelog | The top-level file listing which changeset files to include, and in what order. |
| Changeset | One atomic, uniquely-IDed change — the smallest unit Liquibase tracks and can roll back. |
| DATABASECHANGELOG | Liquibase's own table inside your database — the ledger of what's already been applied. |
| DATABASECHANGELOGLOCK | A lock row preventing two `update` runs from executing against the same DB at once. |
| Rollback | An optional inverse statement per changeset, letting a bad migration be undone without restoring from backup. |

**Real-world use cases**

- **CI/CD-driven migrations** — Wired into a build pipeline so schema changes deploy automatically and in order — no one manually running SQL and hoping nothing was missed.
- **Multi-DB sync** — One RDS cluster hosting several databases replays the same changelog against each, so environments stay structurally identical.
- **Safe rollback** — A bad migration caught minutes after deploy can be reversed with `liquibase rollback`, instead of a full point-in-time restore.

---

## Page 5 · Orchestration & Observability — Jenkins & Grafana

Not yet in the hands-on labs — natural candidates for topics 13 and 14. Jenkins runs the other three tools in order; Grafana watches what they built.

### Jenkins — CI/CD orchestration

> **MENTAL MODEL**
> Jenkins doesn't provision, configure, or migrate anything itself. It calls the tools that do, in a fixed order, whenever a trigger fires — usually a Git push.

```
  Git push ──▶ Webhook ──▶ Jenkins Controller
                                │  schedules
                                ▼
                          Jenkins Agent  (executor)
                                │  runs pipeline stages, in order
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
          terraform apply  ansible-playbook  liquibase update
```

- **Infra-as-code pipeline** — One stage runs Terraform, the next Ansible, the next Liquibase — one controlled sequence instead of three manual runs.
- **Webhook-triggered deploys** — A push to GitHub fires a webhook; Jenkins builds, tests, and deploys automatically.

### Grafana — observability

> **MENTAL MODEL**
> Grafana never writes back to a data source — it's strictly read + alert, which is why it sits outside the loop in the page-1 diagram, not inside it.

```
  Data Sources  (Prometheus, CloudWatch, PostgreSQL exporter)
        │  queried by
        ▼
  Grafana Server
        │  renders
        ▼
  Dashboards (panels)   +   Alert Rules ──▶ Notification (Slack / email)
```

- **Pre-built PG dashboards** — Grafana ships ready-made PostgreSQL dashboards and ~15 alert rules — connections, cache hit ratio, replication status.
- **Unified SRE view** — Metrics from Prometheus, cloud providers, and Postgres land in one dashboard for incident response.

---

## Page 6 · Version Control & the Full Picture — Git & GitOps

Every other tool on this pipeline reads its instructions from files that live in Git.

> **MENTAL MODEL**
> A commit isn't just a save point — for this pipeline, it's the only thing that starts anything. Nobody clicks around the AWS console or SSHes in to hand-edit a config; the only sanctioned way to change infrastructure, config, or schema is to commit to Git, and automation reconciles the real system to match the repo.

```
  Working Directory ── git add ──▶ Staging Area ── git commit ──▶ Local Repo ── git push ──▶ Remote (GitHub)
```

Three local stages before anything leaves your machine — `status` and `diff` exist to inspect each one before it moves on.

| Strategy | Typical fit |
|---|---|
| GitFlow | Structured feature/release/hotfix branches — regulated environments, scheduled releases. |
| Trunk-based | Small changes merged to main constantly — high-velocity teams with strong test coverage. |
| GitHub Flow | Feature branch → PR → merge to main → deploy — a lighter middle ground. |

- **GitOps** — Automation reconciles the real system to match the repo — the repo is always the true current state.
- **PR-gated changes** — A Terraform/Ansible/Liquibase change goes through review before merge, same as application code.

### The whole toolchain, one table

| Tool | Category | Does | Does NOT do |
|---|---|---|---|
| Terraform | Provisioning | Creates/updates/destroys infra to match code | Touch anything inside the OS/DB once it exists |
| Ansible | Config mgmt | Enforces desired state on existing hosts | Create the infrastructure itself |
| Liquibase | Schema VC | Tracks & replays ordered schema changesets | Manage row-level data or infra |
| Jenkins | Orchestration | Runs the above three in the right order | Provision, configure, or migrate anything itself |
| Grafana | Observability | Visualizes metrics, fires alerts | Change anything — read-only |
| Git | Version control | Stores everything; a push/merge starts the pipeline | Run anything by itself |

---

## Page 7 · Knowledge Check — Can I explain this?

Without looking back at pages 1–6. If a box is hard to check, that's the page to re-read.

**Terraform**
- [ ] What's the difference between a resource and a data source?
- [ ] Why does Terraform need a state file at all?
- [ ] What does `plan` do that `apply` doesn't?
- [ ] What causes "drift," and why is it dangerous?

**Ansible**
- [ ] Why is Ansible called "agentless"?
- [ ] What does idempotency actually guarantee?
- [ ] Why can't Ansible run before Terraform in this pipeline?
- [ ] What's the difference between a module and a collection?

**Liquibase**
- [ ] What is a changeset, and why is it the smallest tracked unit?
- [ ] What does `DATABASECHANGELOG` actually store?
- [ ] What happens if you edit an already-applied changeset?

**Jenkins & Grafana**
- [ ] What's the difference between a Jenkins Controller and an Agent?
- [ ] Why does Grafana sit outside the automation loop, not inside it?

**Git & the whole pipeline**
- [ ] What are the three local stages a change passes through before `push`?
- [ ] In GitOps, what's the one sanctioned way to change infrastructure?
- [ ] Without looking at page 1: draw the six-tool pipeline from memory.

---

**Sources referenced while researching this manual** (for further reading, not verbatim quotes):
- HashiCorp — [Terraform Use Cases](https://www.terraform.io/intro/use-cases) & state file documentation
- Red Hat / Ansible — [Ansible Use Cases](https://www.ansible.com/use-cases) & architecture docs
- Liquibase — changelog, changeset & rollback documentation
- Jenkins — pipeline & controller/agent architecture documentation
- Grafana Labs — [Monitor PostgreSQL with Grafana](https://grafana.com/solutions/postgresql/)
- codewithmukesh — GitFlow vs GitHub Flow vs Trunk-Based Development

*PostgresHelp Labs · Build. Operate. Automate. Grow.*
