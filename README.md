# Terraform Task: Provision & Update a 3‑Tier EC2 Setup (ap-south-1)

This repository provisions **three EC2 instances** in **ap-south-1a** using a **single resource block** with `for_each`, along with a Security Group for tiered access. It also demonstrates **updating** the instance type (t2.micro → t3.micro) and **renaming** the Security Group (my-sg → 3-tier-sg), and pushing everything to **GitHub**.

---

## ✨ What You Get

* 3 EC2 instances in **ap-south-1a**:

  * `app-server` (HTTP 80, SSH 22)
  * `db-server` (DB 3306, SSH 22)
  * `proxy-server` (Proxy 8080, SSH 22)
* A single **Security Group** initially named **`my-sg`**, later **renamed to `3-tier-sg`**.
* Variables-driven configuration (**no hardcoding** of region, AMI, key, instance type).
* Clean outputs showing server public IPs and the SG name.

---

## 🧱 Project Structure

```
terraform-task/
 ├── provider.tf
 ├── main.tf
 ├── variables.tf
 ├── outputs.tf
 ├── .gitignore
 └── README.md
```

> The code follows the **simplified roles map** where the map key is the role name and the value is the list of ports.

---

## ✅ Prerequisites

1. **AWS account** with a default VPC in `ap-south-1` (Mumbai).
2. **IAM user/role** with permissions for EC2, VPC, and Security Groups.
3. **AWS credentials** configured locally (any one):

   * `aws configure` (stores under `~/.aws/credentials`)
   * or environment variables: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`
4. **Terraform** v1.4+ installed (`terraform -version`).
5. An **EC2 key pair name** in `ap-south-1` (e.g., `my-keypair`).
6. An **AMI ID** valid for `ap-south-1` (e.g., Amazon Linux 2023). See *Find an AMI* below.

### Find an AMI (example via AWS CLI)

```bash
aws ec2 describe-images \
  --owners amazon \
  --filters 'Name=name,Values=al2023-ami-*' 'Name=architecture,Values=x86_64' \
  --query 'Images | sort_by(@,&CreationDate)[-1].ImageId' \
  --region ap-south-1 --output text
```

Copy the returned AMI ID into your variables.

---

## ⚙️ Variables (no hardcoding)

`variables.tf` uses a **concise** roles map and basic infra inputs:

```hcl
variable "aws_region" { description = "AWS region" type = string default = "ap-south-1" }
variable "availability_zone" { description = "AZ" type = string default = "ap-south-1a" }
variable "instance_type" { description = "Instance type" type = string default = "t2.micro" }
variable "ami_id" { description = "AMI ID" type = string }
variable "key_name" { description = "EC2 key pair name" type = string }

# Map key is the role name; value is the list of allowed ports
variable "server_roles" {
  description = "Map of server roles with required ports"
  type        = map(list(number))
  default = {
    app-server   = [22, 80]
    db-server    = [22, 3306]
    proxy-server = [22, 8080]
  }
}
```

### Optional: `terraform.tfvars` (recommended)

Create a `terraform.tfvars` file (not committed) to store your values:

```hcl
aws_region        = "ap-south-1"
availability_zone = "ap-south-1a"
instance_type     = "t2.micro"   # later update to t3.micro
ami_id            = "ami-xxxxxxxxxxxxxxxxx"
key_name          = "my-keypair"
```

> ⚠️ Never commit secrets or state files. See `.gitignore` below.

---

## 🧩 Provider

`provider.tf`:

```hcl
provider "aws" {
  region = var.aws_region
}
```

---

## 📦 Resources

`main.tf` (key parts):

**Security Group** — starts as `my-sg` and dynamically opens ports used by all roles:

```hcl
# VPC & Subnet Data Sources
data "aws_vpc" "default" { default = true }

data "aws_subnet" "default" {
  availability_zone = var.availability_zone
  default_for_az    = true
}

# Security Group (initially named my-sg)
resource "aws_security_group" "three_tier_sg" {
  name        = "my-sg"               # later change to "3-tier-sg"
  description = "Security group for 3-tier architecture"
  vpc_id      = data.aws_vpc.default.id

  dynamic "ingress" {
    for_each = flatten([
      for role, ports in var.server_roles : [ for port in ports : { port = port } ]
    ])
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Instances — single resource block creates all 3 servers
resource "aws_instance" "servers" {
  for_each              = var.server_roles
  ami                   = var.ami_id
  instance_type         = var.instance_type
  availability_zone     = var.availability_zone
  key_name              = var.key_name
  subnet_id             = data.aws_subnet.default.id
  vpc_security_group_ids = [aws_security_group.three_tier_sg.id]

  tags = {
    Name = each.key         # app-server / db-server / proxy-server
    Role = each.key
  }
}
```

**Outputs** (`outputs.tf`):

```hcl
output "server_ips" {
  description = "Public IPs of the servers"
  value = { for key, s in aws_instance.servers : key => s.public_ip }
}

output "security_group_name" {
  description = "Security group name"
  value       = aws_security_group.three_tier_sg.name
}
```

---

## 🚀 Step‑by‑Step Usage

### 1) Initialize & Validate

```bash
terraform init
terraform fmt -recursive
terraform validate
```

### 2) Review Plan & Apply (Create Phase)

```bash
terraform plan -out create.plan
terraform apply create.plan
```

**What happens:**

* Creates SG **`my-sg`** with ports 22/80/3306/8080 open to `0.0.0.0/0`.
* Launches 3 instances (one per role) in **ap-south-1a**.
* Prints public IPs in outputs.

> 🔍 Verify in AWS Console → EC2 → Instances & Security Groups.

### 3) Update Infrastructure (Task 5 & 6)

* Edit `variables.tf` → change default or set in `terraform.tfvars`:

  ```hcl
  instance_type = "t3.micro"
  ```
* Edit `main.tf` → rename Security Group:

  ```hcl
  resource "aws_security_group" "three_tier_sg" {
    name = "3-tier-sg"
    # ...rest unchanged
  }
  ```

Now apply the updates:

```bash
terraform plan -out update.plan
terraform apply update.plan
```

**What happens:**

* Instances are **updated in-place** to `t3.micro` (no replacement expected).
* SG **name change causes SG replacement**. Terraform will create `3-tier-sg`, attach it to the instances, then destroy `my-sg`.

> ✅ After apply, all instances must show the **new SG `3-tier-sg`** attached.

### 4) Outputs

After each apply, Terraform shows:

```
server_ips = {
  "app-server"   = "x.x.x.x"
  "db-server"    = "y.y.y.y"
  "proxy-server" = "z.z.z.z"
}
security_group_name = "3-tier-sg"
```

---

## 🔎 Verify with AWS CLI (optional)

```bash
# List instances with role & Name tag
aws ec2 describe-instances \
  --filters 'Name=tag:Role,Values=app-server,db-server,proxy-server' \
           'Name=availability-zone,Values=ap-south-1a' \
  --query 'Reservations[].Instances[].[InstanceId,Placement.AvailabilityZone,Tags[?Key==`Name`].Value|[0],PublicIpAddress,InstanceType,SecurityGroups[].GroupName|[0]]' \
  --output table --region ap-south-1

# Check Security Group exists
aws ec2 describe-security-groups \
  --filters 'Name=group-name,Values=3-tier-sg' \
  --query 'SecurityGroups[].{Name:GroupName,Id:GroupId,VpcId:VpcId}' \
  --region ap-south-1 --output table
```

---

## 🧹 Clean Up

```bash
terraform destroy
```

This will terminate the 3 instances and remove the Security Group.

---

## 🗂️ .gitignore (recommended)

```
# Terraform
.terraform/
*.tfstate
*.tfstate.*
crash.log
crash.*.log
*.tfvars
*.tfvars.json
terraform.tfplan
.terraform.lock.hcl

# OS/Misc
.DS_Store
```

> Commit a `terraform.tfvars.example` (no secrets) to show other users the expected inputs.

---

## 🧭 GitHub: Create & Push Repo

```bash
git init
git branch -M main
git remote add origin git@github.com:<your-username>/terraform-task.git

git add .
git commit -m "Provision 3-tier EC2 with SG; updates to t3.micro and 3-tier-sg"

git push -u origin main
```

Open your repository and confirm files are visible.

---

## 🧰 Troubleshooting

* **No default VPC/subnet in ap-south-1a**: Create a default VPC (or specify `vpc_id` and a subnet explicitly) and adjust the `data aws_subnet` to target your subnet by ID.
* **AMI not found/launch fails**: Ensure the AMI ID is valid in `ap-south-1` and matches your architecture.
* **Key pair not found**: Create/import a key pair in `ap-south-1` and set `key_name` accordingly.
* **SG name change**: This triggers **replacement**. Terraform will handle re-attachment; if it fails, run `terraform apply` again after checking EC2 limits/pending states.
* **Provider version constraints**: If needed, pin versions in a `required_providers` block.

---

## 🔒 Security Notes

* Ingress is currently open to the world (`0.0.0.0/0`) for demo simplicity. Restrict to your **office IP / VPN** in production.
* Store credentials safely; prefer **short‑lived roles** (e.g., SSO) over static keys.

---

## ➕ Optional Enhancements

* **Lookup latest AMI** via `data "aws_ami"` instead of passing a var.
* **Add `lifecycle { create_before_destroy = true }`** to SG if you want to avoid brief attachment gaps during rename.
* **Per‑role user\_data** to bootstrap app/db/proxy.
* **Remote backend** (S3 + DynamoDB) for team collaboration and locking.

---

## 📄 License

MIT (or your preferred license).
