## Documentation

Move initial files
```
cd C:\Users\hp\Documents\Course\ansible-postgresql

Remove-Item -Recurse -Force .git

git init

git config user.name "Y_USER_NAME"
git config user.email "Y_USER_EMAIL"

git remote add origin https://github.com/postgreshelp/NewTerraform.git

git add .

git status

git commit -m "Add Terraform VPC read and import examples"

git branch -M master

git push -u origin master
```



### start with empty folder

#### 1. Find/read the existing default VPC.
#### create providers.tf

```
terraform {
	required_providers {
		aws = {
			source  = "hashicorp/aws"
			version = "6.63.0"
		}
	}
}

provider "aws" {
	region = "us-east-1"
}
```

#### create data.tf

```
data "aws_vpc" "default" {
  default = true
}
```

#### create outputs.tf

```
output "default_vpc_id" {
  description = "ID of the existing default VPC"
  value       = data.aws_vpc.default.id
}
```

#### Run the below commands

```
Terraform init
Terraform validate
Terraform apply
```

#### Sample output

```
PS C:\Users\hp\Documents\Course\NewTerraform> Terraform init
Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.100.0...
- Installed hashicorp/aws v5.100.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
PS C:\Users\hp\Documents\Course\NewTerraform> 
PS C:\Users\hp\Documents\Course\NewTerraform> terraform validate
Success! The configuration is valid.

PS C:\Users\hp\Documents\Course\NewTerraform> 
PS C:\Users\hp\Documents\Course\NewTerraform> terraform apply
data.aws_vpc.default: Reading...
data.aws_vpc.default: Read complete after 2s [id=vpc-063618af923af9f03]

Changes to Outputs:
  + default_vpc_id = "vpc-063618af923af9f03"

You can apply this plan to save these new output values to the Terraform state,
without changing any real infrastructure.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes


Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

default_vpc_id = "vpc-063618af923af9f03"
PS C:\Users\hp\Documents\Course\NewTerraform> 
````

### 2. Import an existing VPC into Terraform

### Get the CIDR for your VPC

aws ec2 describe-vpcs --vpc-ids vpc-02f476ec787e998c6

expected output
```
10.10.0.0/16
```

create aws-default-import.tf

```
resource "aws_vpc" "default-import" {
  cidr_block = "10.10.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "default-import"
  }
}
```

Run the below commands

```
Terraform init
Terraform validate
terraform import aws_vpc.default-import vpc-02f476ec787e998c6
```

###output log

```
PS C:\Users\hp\Documents\Course\NewTerraform> terraform init   
Initializing the backend...

Initializing provider plugins...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Using previously-installed hashicorp/aws v5.100.0

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
PS C:\Users\hp\Documents\Course\NewTerraform> terraform validate
Success! The configuration is valid.

PS C:\Users\hp\Documents\Course\NewTerraform> 
PS C:\Users\hp\Documents\Course\NewTerraform> 
PS C:\Users\hp\Documents\Course\NewTerraform> terraform import aws_vpc.b03_vpc vpc-02f476ec787e998c6
aws_vpc.b03_vpc: Importing from ID "vpc-02f476ec787e998c6"...
data.aws_vpc.default: Reading...
aws_vpc.b03_vpc: Import prepared!
  Prepared aws_vpc for import
aws_vpc.b03_vpc: Refreshing state... [id=vpc-02f476ec787e998c6]
data.aws_vpc.default: Read complete after 2s [id=vpc-063618af923af9f03]

Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.

PS C:\Users\hp\Documents\Course\NewTerraform> 
```
#### Push everything to get

### To remove everything from git

```
cd C:\Users\hp\Documents\Course\NewTerraform
Remove-Item -Recurse -Force .git
```

#### 1. Right-click NewTerraform (the folder/root at the top of Explorer) → New File.

```
.gitignore
```

Then put this inside it:

```
.terraform/
*.tfstate
*.tfstate.*
.terraform.tfstate.lock.info
*.code-workspace
```

Now run below commands

```

git add .

git status

git commit -m "Add Terraform VPC read and import examples"

git push -u origin master

```

Sample log

```
PS C:\Users\hp\Documents\Course\NewTerraform>
PS C:\Users\hp\Documents\Course\NewTerraform> cd C:\Users\hp\Documents\Course\NewTerraform
PS C:\Users\hp\Documents\Course\NewTerraform> Remove-Item -Recurse -Force .git
Remove-Item: Cannot find path 'C:\Users\hp\Documents\Course\NewTerraform\.git' because it does not exist.
PS C:\Users\hp\Documents\Course\NewTerraform> git init
Initialized empty Git repository in C:/Users/hp/Documents/Course/NewTerraform/.git/
PS C:\Users\hp\Documents\Course\NewTerraform> git config user.name "postgreshelp"
PS C:\Users\hp\Documents\Course\NewTerraform> git config user.email "postgreshelp@gmail.com"
PS C:\Users\hp\Documents\Course\NewTerraform> git remote add origin https://github.com/postgreshelp/NewTerraform.git
PS C:\Users\hp\Documents\Course\NewTerraform> git add .
PS C:\Users\hp\Documents\Course\NewTerraform> git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   .gitignore
        new file:   .terraform.lock.hcl
        new file:   Documentation.md
        new file:   aws-vpc.tf
        new file:   data.tf
        new file:   outputs.tf
        new file:   providers.tf

PS C:\Users\hp\Documents\Course\NewTerraform> git remote add origin https://github.com/postgreshelp/NewTerraform.git
error: remote origin already exists.
PS C:\Users\hp\Documents\Course\NewTerraform>
PS C:\Users\hp\Documents\Course\NewTerraform> git commit -m "Add Terraform VPC read and import examples"
[master (root-commit) 6e6d6c0] Add Terraform VPC read and import examples
 7 files changed, 258 insertions(+)
 create mode 100644 .gitignore
 create mode 100644 .terraform.lock.hcl
 create mode 100644 Documentation.md
 create mode 100644 aws-vpc.tf
 create mode 100644 data.tf
 create mode 100644 outputs.tf
 create mode 100644 providers.tf
PS C:\Users\hp\Documents\Course\NewTerraform> git branch -M master
PS C:\Users\hp\Documents\Course\NewTerraform> git push -u origin master
Enumerating objects: 9, done.
Counting objects: 100% (9/9), done.
Delta compression using up to 8 threads
Compressing objects: 100% (8/8), done.
Writing objects: 100% (9/9), 3.31 KiB | 1.66 MiB/s, done.
Total 9 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To https://github.com/postgreshelp/NewTerraform.git
 * [new branch]      master -> master
branch 'master' set up to track 'origin/master'.
PS C:\Users\hp\Documents\Course\NewTerraform>
```

#### create a new vpc

### aws-create-vpc.tf

```
resource "aws_vpc" "bt01-vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "bt01-vpc"
  }
}

```

### outputs.tf
```
output "aws_vpc" {
  description = "ID of the newly created VPC"
  value       = aws_vpc.bt01-vpc.id
}
```

### run below commands

```
terraform fmt
terraform validate
terraform plan
 ```

 plan should show
 ```
 Plan: 1 to add, 0 to change, 0 to destroy.
 ```

```
terraform apply
```

### now git push

git status
git add .
git commit -m "Add a new VPC"
git push -u origin master


### add aws-networking.tf

```
resource "aws_internet_gateway" "bt01-igw" {
  vpc_id = aws_vpc.bt01-vpc.id

  tags = {
    Name = "bt01-igw"
  }
}

resource "aws_subnet" "bt01-public-subnet" {
  vpc_id                  = aws_vpc.bt01-vpc.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "bt01-public-subnet"
  }
}

resource "aws_subnet" "bt01-private-subnet" {
  vpc_id            = aws_vpc.bt01-vpc.id
  availability_zone = "us-east-1b"
  cidr_block        = "10.0.2.0/24"

  tags = {
    Name = "bt01-private-subnet"
  }
}

resource "aws_route_table" "bt01-route-table" {
  vpc_id = aws_vpc.bt01-vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.bt01-igw.id
  }

  tags = {
    Name = "bt01-route-table"
  }
}

resource "aws_route_table_association" "bt01-public-subnet-association" {
  subnet_id      = aws_subnet.bt01-public-subnet.id
  route_table_id = aws_route_table.bt01-route-table.id
}

resource "aws_route_table_association" "bt01-private-subnet-association" {
  subnet_id      = aws_subnet.bt01-private-subnet.id
  route_table_id = aws_route_table.bt01-route-table.id
}
```

Apply changes

```
terraform fmt
terraform validate
terraform plan
```

sample output

```
Plan: 5 to add, 0 to change, 0 to destroy.
```

```
terraform apply
```

### apply aws-aurora.tf

```
##main content
resource "aws_db_subnet_group" "bt01_aurora" {
  name = "bt01-aurora-subnet-group"

  subnet_ids = [
    aws_subnet.bt01-public-subnet.id,
    aws_subnet.bt01-private-subnet.id
  ]

  tags = {
    Name = "bt01-aurora-subnet-group"
  }
}
##define security groups
resource "aws_security_group" "bt01_aurora" {
  name        = "bt01-aurora-sg"
  description = "Security group for Aurora PostgreSQL"
  vpc_id      = aws_vpc.bt01-vpc.id

  ingress {
    description = "PostgreSQL"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "bt01-aurora-sg"
  }
}
## define aurora database
resource "aws_rds_cluster" "bt01_aurora" {
  cluster_identifier = "bt01-aurora"
  engine             = "aurora-postgresql"

  master_username = "postgres"
  master_password = "postgres"

  db_subnet_group_name   = aws_db_subnet_group.bt01_aurora.name
  vpc_security_group_ids = [aws_security_group.bt01_aurora.id]

  database_name = "paylite"

  skip_final_snapshot = true

  tags = {
    Name = "bt01-aurora"
  }
}

resource "aws_rds_cluster_instance" "bt01_aurora" {
  identifier         = "bt01-aurora-instance-1"
  cluster_identifier = aws_rds_cluster.bt01_aurora.id

  instance_class = "db.t3.medium"
  engine         = aws_rds_cluster.bt01_aurora.engine

  tags = {
    Name = "bt01-aurora-instance-1"
  }
}



```

```
Apply changes

```
terraform fmt
terraform validate
terraform plan
```
```
