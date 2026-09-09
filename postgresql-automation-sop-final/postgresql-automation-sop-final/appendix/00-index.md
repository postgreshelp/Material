# Course Index

**PostgresHelp Labs**

Terraform → Ansible → Liquibase → (Jenkins, Grafana) → Git, built on a Postgres Aurora cluster. Every page below follows the same rhythm: mental model first, then why a command matters, then the lab, then how to read what happened.

## How a learning unit is structured

`Mental model` → `Why this command?` → `Lab` → `Expected output` → `What changed?` → `Verify` → `Can I explain it?`

---

## 00 · Orientation

**00 — This index**
*You are here.* The map. Six documents, each covering a different layer of the same project — start with Theory if you're new, or jump straight to a Lab if you're picking up where you left off.

## 01 · Theory

**01 — [The toolchain: what each tool is actually for](13-theory-manual-toolchain.md)**
*7-page theory manual.* What each of the six tools is, its architecture, and where it deliberately stops and hands off to the next one. Read this before any lab — it's the mental model everything else builds on.

## 02–04 · Concepts + Lab

**02 — [Terraform: VPC & Aurora](01-terraform-vpc-aurora-reference.md)**
*Concepts + Lab.* Five sub-labs — read the default VPC, create a new one, wire up networking, stand up Aurora, and import an existing VPC (with a real troubleshooting case on why import ≠ config match).

**03 — [Environment setup, steps 02–08](02-08-windows-setup-guide.md)**
*Lab · Windows workstation.* VS Code, Git, GitHub login, Copilot, AWS CLI, AWS authentication, Terraform install — each with the mental model, the why, and how to read the result.

**04 — [Environment setup, steps 09–12](09-12-liquibase-ansible-verify.md)**
*Lab · Linux host.* Liquibase, Ansible, Java/PostgreSQL prerequisites, and a final end-to-end verification across everything installed in 02–11.

## 05 · Reference

**05 — [Every command, by tool](14-command-cheatsheet.md)**
*Command cheatsheet.* Pure lookup — no concepts taught here. Git, Terraform, AWS CLI, Ansible, Liquibase, psql, and a PowerShell section for auditing what's actually installed on Windows.

---

## Reading this course

1. **Mental model before commands.** Every topic opens with what's actually going on, not a command to type.
2. **Why before important commands.** A command you understand the purpose of is one you won't forget.
3. **Expected output, not just output.** Each key command says what to look for and what failure means.
4. **What changed, after every lab.** A before/after diagram connects the command to the system state it produced.
5. **Can I explain it?** Every learning unit ends with a checklist — the real test is reproducing it without the doc.

---
*PostgresHelp Labs · Build. Operate. Automate. Grow.*
