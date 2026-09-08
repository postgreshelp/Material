# PostgreSQL Automation Project --- Architecture

## 1. Final Conceptual Architecture

The project is designed around one automation/control EC2 instance and
PostgreSQL environments.

``` mermaid
flowchart TB
    DEV[Developer / DBA]
    GIT[GitHub\nSource Repository]

    EC2[EC2 Automation Server\nAnsible + Liquibase + Jenkins\nTerraform CLI]

    AWS[AWS Infrastructure]

    VPC[VPC]
    NET[Subnets / Routing / Security]
    DBDEV[(PostgreSQL DEV)]
    DBTEST[(PostgreSQL TEST)]
    DBPROD[(PostgreSQL PROD)]

    DEV -->|git push| GIT
    GIT -->|clone / pull| EC2

    EC2 -->|Terraform| AWS
    AWS --> VPC
    VPC --> NET

    EC2 -->|Ansible| DBDEV
    EC2 -->|Ansible| DBTEST
    EC2 -->|Ansible| DBPROD

    EC2 -->|Liquibase| DBDEV
    EC2 -->|Liquibase| DBTEST
    EC2 -->|Liquibase| DBPROD

    EC2 -->|Jenkins orchestrates| EC2
```

------------------------------------------------------------------------

# 2. Responsibility Architecture

``` mermaid
flowchart LR
    G[Git]
    T[Terraform]
    A[Ansible]
    L[Liquibase]
    J[Jenkins]
    D[(PostgreSQL)]

    G -->|versioned files| T
    G -->|versioned playbooks| A
    G -->|versioned changelogs| L
    G -->|pipeline definition| J

    T -->|provision| D
    A -->|configure / administer| D
    L -->|schema changes| D

    J -->|orchestrate| T
    J -->|orchestrate| A
    J -->|orchestrate| L
```

------------------------------------------------------------------------

# 3. Infrastructure Architecture

The initial hands-on lab deliberately uses a simple AWS network.

``` mermaid
flowchart TB
    AWS[AWS]
    VPC[VPC\n10.0.0.0/16]

    IGW[Internet Gateway]
    RT[Route Table]

    SUB1[Subnet 1\n10.0.1.0/24\nus-east-1a]
    SUB2[Subnet 2\n10.0.2.0/24\nus-east-1b]

    DBSG[DB Subnet Group]
    SG[Security Group\nTCP 5432]

    AURORA[(Aurora PostgreSQL\nPayLite)]

    AWS --> VPC
    VPC --> IGW
    VPC --> RT
    RT --> SUB1
    RT --> SUB2
    SUB1 --> DBSG
    SUB2 --> DBSG
    DBSG --> AURORA
    SG --> AURORA
```

### Lab simplification

The subnet named `private` in the lab is intentionally simple. If it
shares a route to an Internet Gateway, it is not technically a private
subnet.

For a production architecture, use proper private subnets and
appropriate routing.

------------------------------------------------------------------------

# 4. Automation Server Architecture

``` mermaid
flowchart TB
    EC2[EC2 Automation Server]

    TF[Terraform CLI]
    AN[Ansible]
    LB[Liquibase]
    JK[Jenkins]
    GIT[Git]

    TF --> EC2
    AN --> EC2
    LB --> EC2
    JK --> EC2
    GIT --> EC2

    EC2 --> AWS[AWS APIs]
    EC2 --> PG[(PostgreSQL over TCP 5432)]
```

The EC2 machine is the execution point in the lab.

RDS/Aurora is not treated as an SSH-managed Ansible host.

------------------------------------------------------------------------

# 5. Ansible-to-PostgreSQL Architecture

``` mermaid
sequenceDiagram
    participant J as Jenkins / User
    participant A as Ansible on EC2
    participant M as community.postgresql module
    participant P as PostgreSQL / Aurora

    J->>A: Run playbook
    A->>M: Execute PostgreSQL task
    M->>P: TCP connection :5432
    P-->>M: Result
    M-->>A: Task result
    A-->>J: Play recap
```

Inventory:

``` ini
[postgresql]
localhost ansible_connection=local
```

The inventory describes the Ansible execution target.

The PostgreSQL connection is defined separately in variables/module
parameters.

------------------------------------------------------------------------

# 6. Liquibase Architecture

``` mermaid
flowchart LR
    G[Git Repository]
    M[Master Changelog]
    C1[001-create-schema.sql]
    C2[002-create-table.sql]
    LB[Liquibase]
    HIST[DATABASECHANGELOG]
    PG[(PostgreSQL)]

    G --> M
    M --> C1
    M --> C2
    M --> LB
    LB --> PG
    LB --> HIST
```

The important concept is:

``` text
Changelog files
      |
      v
Liquibase
      |
      +----> Apply changes
      |
      +----> Track changes
      |
      +----> Validate changes
      |
      v
PostgreSQL
```

------------------------------------------------------------------------

# 7. Jenkins Orchestration Architecture

``` mermaid
flowchart LR
    G[Git Push]
    J[Jenkins]

    S1[Stage 1\nVerify]
    S2[Stage 2\nTerraform]
    S3[Stage 3\nAnsible]
    S4[Stage 4\nLiquibase]
    S5[Stage 5\nMonitoring]

    G --> J
    J --> S1
    S1 --> S2
    S2 --> S3
    S3 --> S4
    S4 --> S5
```

Jenkins controls **when and in what order** the tools execute.

------------------------------------------------------------------------

# 8. Complete End-to-End Flow

``` mermaid
flowchart TB
    DEV[Developer]
    GIT[GitHub]

    J[Jenkins]
    TF[Terraform]
    AN[Ansible]
    LB[Liquibase]
    MON[monitoring2.sh]

    INF[AWS Infrastructure]
    PG[(PostgreSQL)]

    DEV -->|git push| GIT
    GIT -->|checkout| J

    J --> TF
    TF --> INF

    J --> AN
    AN --> PG

    J --> LB
    LB --> PG

    J --> MON
    MON --> PG
```

------------------------------------------------------------------------

# 9. Environment Promotion

The intended production-style concept is:

``` mermaid
flowchart LR
    DEV[(DEV)]
    TEST[(TEST)]
    PROD[(PROD)]

    CHANGE[Versioned DB Change]

    CHANGE --> DEV
    DEV -->|validate / approve| TEST
    TEST -->|validate / approve| PROD
```

Git identifies the version of the change.

Liquibase controls database change application.

Jenkins controls the promotion workflow.

------------------------------------------------------------------------

# 10. What Runs Where?

  -----------------------------------------------------------------------
  Component                           Location
  ----------------------------------- -----------------------------------
  VS Code                             Local machine

  Git                                 Local + GitHub

  Terraform CLI                       Local during the learning phase;
                                      can later run in Jenkins

  Terraform state                     Terraform-managed state location

  Ansible                             EC2 automation server

  Liquibase                           EC2 automation server

  Jenkins                             EC2 automation server in this lab

  PostgreSQL                          AWS RDS/Aurora

  monitoring2.sh                      EC2 automation server
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 11. Final Mental Model

``` text
                    DEVELOPER
                        |
                        | git push
                        v
                 +-------------+
                 |   GITHUB    |
                 +-------------+
                        |
                        | checkout / pull
                        v
             +-----------------------+
             |   EC2 AUTOMATION      |
             |                       |
             | Jenkins               |
             | Terraform CLI         |
             | Ansible               |
             | Liquibase              |
             | monitoring2.sh        |
             +-----------------------+
                 |       |       |
             Terraform Ansible Liquibase
                 |       |       |
                 v       v       v
              AWS     PostgreSQL Schema
           Infrastructure Operations Changes
                         |
                         v
                 +---------------+
                 | PostgreSQL    |
                 | DEV/TEST/PROD |
                 +---------------+
```

The central architectural principle is:

> **Git stores the desired version, specialized tools perform
> specialized work, and Jenkins orchestrates the sequence.**
