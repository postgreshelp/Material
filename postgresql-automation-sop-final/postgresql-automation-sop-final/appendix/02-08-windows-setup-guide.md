# 03 · Lab — Windows Workstation Setup (02–08)

**PostgresHelp Labs**

Generic, repeatable install and verification steps for the tools used through the rest of this course — VS Code, Git, GitHub, Copilot, AWS CLI, and Terraform. Each section explains *why* the step matters before showing the command, then how to read the result.

---

## 02 · VS Code Setup

Install Visual Studio Code and the extensions used for Terraform, Ansible/YAML, and SQL editing.

> **MENTAL MODEL**
> VS Code itself is just a text editor. Every tool-specific capability — Terraform syntax highlighting, Copilot completions — comes from an extension, installed separately. "Installing VS Code" and "being ready to write Terraform" are two different checkpoints.

> **BEFORE YOU START**
> - ✓ Windows 10/11, admin rights on the machine
> - ✓ Stable internet connection, ~1 GB free disk space

**Extensions to install:** HashiCorp Terraform · YAML (Red Hat) · GitLens · GitHub Copilot (topic 05)

**Why these four?** Each one matches a file type you'll actually edit in this course: `.tf`, `.yml`, and Git history — not a generic "useful extensions" list.

**Steps**
1. Download and run the installer — code.visualstudio.com → Windows (User Installer). Accept the license, keep defaults, and check **"Add to PATH"** — this lets you run `code` from any terminal.
2. Install the extensions from the command line.

> **WHY THIS COMMAND?** Doing this via CLI instead of clicking through the Extensions panel means you can paste the same three lines onto a second machine and get an identical setup — no memory required.

```powershell
PS C:\Users\hp> code --install-extension hashicorp.terraform
PS C:\Users\hp> code --install-extension redhat.vscode-yaml
PS C:\Users\hp> code --install-extension eamodio.gitlens
```

> **EXPECTED RESULT**
> - **Look for:** a version/commit/architecture triple, three lines, no error text.
> - **If it fails:** "not recognized" means the installer's "Add to PATH" step was skipped — reinstall and check that box.

```powershell
PS C:\Users\hp> code --version
1.94.2
5a0e0b3...
x64
```

---

## 03 · Git Installation & Configuration

Install Git for Windows and set your identity so commits are attributed correctly.

> **MENTAL MODEL**
> "Installed" and "configured" are two separate checkpoints. Git works the moment it's installed — but every commit needs an author, and Git won't invent one for you. Skipping config just defers the error to your first commit instead of catching it now.

> **BEFORE YOU START**
> - ✓ VS Code installed (topic 02) — not required for Git itself, but you'll want an editor open next

**Steps**
1. Download and install Git for Windows — git-scm.com/download/win. Keep the default editor unless you have a preference, and select **"Git from the command line and also from 3rd-party software"** for the PATH option.
2. Set your identity.

> **WHY THIS COMMAND?** Every commit needs a name and email — this is what shows up in `git log` and on GitHub. `--global` means every repo on this machine, not just this one.

```powershell
PS C:\Users\hp> git config --global user.name "Y_USER_NAME"
PS C:\Users\hp> git config --global user.email "Y_USER_EMAIL"
```

> **EXPECTED RESULT**
> - **git config --list:** shows `user.name=...` and `user.email=...` echoing back what you just set.
> - **Empty output?** You forgot `--global`, or you're inside a repo with its own local override.

```powershell
PS C:\Users\hp> git config --list
user.name=Y_USER_NAME
user.email=Y_USER_EMAIL
core.autocrlf=true
```
```powershell
PS C:\Users\hp> git --version
git version 2.47.0.windows.1
```

---

## 04 · GitHub Login

Authenticate Git and VS Code against your GitHub account so pushes don't prompt for a password every time.

> **MENTAL MODEL**
> Git identity (topic 03: who wrote this commit) and GitHub authentication (this topic: who's allowed to push it) are unrelated. You can have a perfectly configured name/email and still be rejected on push if you've never authenticated.

> **BEFORE YOU START**
> - ✓ Git installed and identity configured (topic 03)
> - ✓ A GitHub account, and a remote already added via `git remote add origin`

**Steps**
1. Sign in from VS Code — click the account icon (bottom-left) → **Sign in with GitHub**. This opens your browser for OAuth consent, then hands a token back to VS Code and to Git Credential Manager.
2. First push triggers the credential prompt.

> **WHY THIS COMMAND?** `git push -u` does two things at once: pushes the commits, and remembers this branch's remote pairing so future `push`/`pull` need no arguments.

```powershell
PS C:\Users\hp> git push -u origin master
info: please complete authentication in your browser...
Enumerating objects: 9, done.
...
branch 'master' set up to track 'origin/master'.
```

> **EXPECTED RESULT**
> - **Look for:** "Hi Y_USER_NAME! You've successfully authenticated"
> - **What it means:** Your local SSH/token identity is now trusted by GitHub — not the same as Git commit identity from topic 03.

```powershell
PS C:\Users\hp> ssh -T git@github.com
Hi Y_USER_NAME! You've successfully authenticated, but GitHub does not provide shell access.
```

---

## 05 · GitHub Copilot Setup

Copilot piggybacks on the GitHub login from step 04 — no separate credentials needed.

> **MENTAL MODEL**
> One login, three tools: VS Code's auth session → GitHub OAuth token → Copilot extension reuses that same token → Copilot's backend validates your subscription against it. There's nothing to separately "log into" for Copilot.

> **BEFORE YOU START**
> - ✓ Signed into GitHub from VS Code (topic 04)
> - ✓ An active Copilot subscription or trial on your GitHub account

**Steps**
1. Install the extensions.
```powershell
PS C:\Users\hp> code --install-extension GitHub.copilot
PS C:\Users\hp> code --install-extension GitHub.copilot-chat
```
2. Reload — it should just work. Because you're already signed into GitHub, Copilot picks up the same session automatically. If not, click the Copilot icon in the status bar → **Sign in to use Copilot**.

> **EXPECTED RESULT**
> - **Look for:** a greyed-out inline suggestion within a second or two of typing a comment in any code file.
> - **Nothing appears?** Check the Copilot status-bar icon for an error badge — usually means the subscription check failed, not the extension.

---

## 06 · AWS CLI Setup

Install the AWS CLI v2 — Terraform's AWS provider can use its config, and it's needed for verification commands throughout this course.

> **MENTAL MODEL**
> Installing the CLI and authenticating it (topic 07) are two different steps — this topic only gets you the `aws` command itself. Running `aws` anything before topic 07 will fail on credentials, not on installation.

> **BEFORE YOU START**
> - ✓ Admin rights to run the MSI installer

**Steps**
1. Download and run the MSI installer — `awscli.amazonaws.com/AWSCLIV2.msi`. Accept defaults — it installs to `C:\Program Files\Amazon\AWSCLIV2` and adds itself to PATH.
2. Open a new terminal. PATH changes only apply to new shells — close and reopen PowerShell/VS Code terminal before verifying.

> **EXPECTED RESULT**
> - **Look for:** an `aws-cli/2.x` version string.

```powershell
PS C:\Users\hp> aws --version
aws-cli/2.19.0 Python/3.12.6 Windows/10 exe/AMD64 prompt/off
```

---

## 07 · AWS Authentication

Store credentials locally so both the AWS CLI and Terraform's AWS provider can find them.

> **MENTAL MODEL**
> Terraform never asks you for AWS keys directly — it delegates entirely to the AWS SDK's standard credential chain. Whatever `aws configure` writes to disk here is the same file Terraform reads later. Get this step right and topic 01 "just works"; get it wrong and every Terraform error will look like a Terraform problem when it's actually this.

> **BEFORE YOU START**
> - ✓ AWS CLI installed (topic 06)
> - ✓ An IAM user with programmatic access enabled in the AWS Console

**Steps**
1. Create an IAM access key — AWS Console → IAM → your user → Security credentials → **Create access key**. Choose "Command Line Interface (CLI)." Save the Access Key ID and Secret Access Key shown once.
2. Run `aws configure`.

> **WHY THIS COMMAND?** This writes the [default] profile to `%USERPROFILE%\.aws\credentials` and `...\.aws\config` — the exact files the credential chain below reads.

```powershell
PS C:\Users\hp> aws configure
AWS Access Key ID [None]: AKIA................
AWS Secret Access Key [None]: ****************************
Default region name [None]: us-east-1
Default output format [None]: json
```

**Credential resolution chain**
```
terraform apply
      │
      ▼
Terraform AWS Provider
      │
      ▼
AWS SDK credential chain
      │
      ├── Environment variables? (AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY)
      │       └── none set
      │
      ├── --profile flag or AWS_PROFILE env var?
      │       └── none set → falls back to "default"
      │
      ├── Shared credentials file?
      │       └── %USERPROFILE%\.aws\credentials
      │
      ▼
C:\Users\hp\.aws\credentials
      │
      ▼
[default]
aws_access_key_id = AKIA................
aws_secret_access_key = ****************************
      │
      ▼
AWS API (signed request)
```

> **Security note** — the credentials file is plaintext on disk. Never commit `.aws/` to git, and prefer short-lived SSO credentials (`aws configure sso`) over long-lived access keys where supported.

> **EXPECTED RESULT**
> - **Look for:** a JSON block with `Account` and `Arn` — this is "who am I," not "what can I do."
> - **If it fails:** "Unable to locate credentials" means `aws configure` either wasn't run or wrote to a profile you're not using.

```powershell
PS C:\Users\hp> aws sts get-caller-identity
{
    "UserId": "AIDAEXAMPLE...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/hp"
}
```

---

## 08 · Terraform Installation

Follow HashiCorp's official Windows installation docs — summarized here for reference.

> **MENTAL MODEL**
> Terraform itself is just a single binary with no runtime dependency — it doesn't need AWS credentials to install or exist. Everything from topic 07 only matters the moment you run `terraform apply` against a real provider.

> **BEFORE YOU START**
> - ✓ Admin rights, or a package manager (choco/winget) already installed

**Steps**
1. Get the binary — developer.hashicorp.com/terraform/install, or via a package manager:
```powershell
PS C:\Users\hp> choco install terraform
# or
PS C:\Users\hp> winget install HashiCorp.Terraform
```
2. Add to PATH (manual installs only) — unzip `terraform.exe` into a folder (e.g. `C:\terraform`) and add that folder to your System PATH. Package-manager installs do this automatically.

> **Reference** — full official steps, including checksum verification, are in HashiCorp's docs: `developer.hashicorp.com/terraform/install`.

> **EXPECTED RESULT**
> - **Look for:** "Terraform v1.9.x on windows_amd64" — this is topic 01's actual starting point.

```powershell
PS C:\Users\hp> terraform -version
Terraform v1.9.8
on windows_amd64
```

---

## Can I explain this? (topics 02–08)

- [ ] What's the difference between Git identity (03) and GitHub authentication (04)?
- [ ] Why does Copilot need no separate login of its own?
- [ ] Which file does `aws configure` write to, and who else reads it?
- [ ] Why would a Terraform auth error in topic 01 actually be a topic 07 problem?
- [ ] What does `-u` on `git push` actually set up for future pushes?
- [ ] Without looking — list all 7 tools from 02–08 in the order they were installed, and why that order matters.

---
*PostgresHelp Labs · Build. Operate. Automate. Grow.*
