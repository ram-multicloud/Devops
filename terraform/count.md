# Terraform `count` — Real Examples & Conditional Logic

## 1. What is `count`?

`count` is a meta-argument that creates multiple instances of a resource, data source, or module from a single block, based on a whole number.

```hcl
resource "aws_instance" "server" {
  count = 3

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "server-${count.index}"
  }
}
```

---

## 2. Real-World Example 1: Creating Multiple Subnets

```hcl
variable "subnet_cidrs" {
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

variable "azs" {
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_subnet" "this" {
  count = length(var.subnet_cidrs)

  vpc_id            = var.vpc_id
  cidr_block        = var.subnet_cidrs[count.index]
  availability_zone = var.azs[count.index]

  tags = {
    Name = "subnet-${count.index}"
  }
}
```

## 3. Real-World Example 2: Creating IAM Users from a List

```hcl
variable "iam_usernames" {
  default = ["alice", "bob", "carol"]
}

resource "aws_iam_user" "users" {
  count = length(var.iam_usernames)
  name  = var.iam_usernames[count.index]
}
```

## 4. Real-World Example 3: Attaching EBS Volumes to Multiple Instances

```hcl
resource "aws_instance" "app" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

resource "aws_ebs_volume" "data" {
  count             = 3
  availability_zone = aws_instance.app[count.index].availability_zone
  size              = 20
}

resource "aws_volume_attachment" "attach" {
  count       = 3
  device_name = "/dev/sdh"
  volume_id   = aws_ebs_volume.data[count.index].id
  instance_id = aws_instance.app[count.index].id
}
```

---

## 5. `count` with Conditions (if / else-if style)

Terraform's HCL has **no native `if`/`elseif`/`else` statement**. Instead, conditional logic is expressed with the **ternary operator**:

```
condition ? true_value : false_value
```

Since `count` only accepts a number, you combine ternary expressions with booleans (`0` = don't create, `1` = create) to simulate `if`/`else` behavior. Nested ternaries simulate `elseif`.

### Example A: Simple `if` (create or not)

```hcl
variable "create_instance" {
  type    = bool
  default = true
}

resource "aws_instance" "optional" {
  count = var.create_instance ? 1 : 0

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```
*If `create_instance = false` → 0 instances created.*

### Example B: `if / else` — different counts per environment

```hcl
variable "environment" {
  default = "dev"
}

resource "aws_instance" "app" {
  count = var.environment == "prod" ? 5 : 1

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```
*If `environment == "prod"` → 5 instances, else → 1 instance.*

### Example C: `if / elseif / else` — nested ternary

```hcl
variable "environment" {
  default = "staging"
}

resource "aws_instance" "app" {
  count = (
    var.environment == "prod"    ? 5 :
    var.environment == "staging" ? 3 :
    var.environment == "dev"     ? 1 : 0
  )

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

Logic breakdown (equivalent pseudocode):
```
if environment == "prod":
    count = 5
elseif environment == "staging":
    count = 3
elseif environment == "dev":
    count = 1
else:
    count = 0
```

### Example D: Combining multiple conditions (AND / OR)

```hcl
variable "environment"     { default = "prod" }
variable "enable_backup"   { default = true }

resource "aws_instance" "backup_node" {
  count = (var.environment == "prod" && var.enable_backup) ? 2 : 0

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```
*Creates 2 instances only if BOTH conditions are true.*

### Example E: Conditional module creation

```hcl
variable "deploy_monitoring" {
  default = true
}

module "monitoring" {
  source = "./modules/monitoring"
  count  = var.deploy_monitoring ? 1 : 0
}
```

Since a module with `count` becomes a list, reference it as:
```hcl
output "monitoring_arn" {
  value = var.deploy_monitoring ? module.monitoring[0].arn : null
}
```

---

## 6. Quick Reference Table

| Goal | Pattern |
|---|---|
| Create or skip a resource | `count = condition ? 1 : 0` |
| Different count per condition | `count = condition ? A : B` |
| Multiple conditions (elseif) | Nested ternary chain |
| AND condition | `count = (cond1 && cond2) ? N : 0` |
| OR condition | `count = (cond1 \|\| cond2) ? N : 0` |
| Count from a list length | `count = length(var.list)` |

---

## 7. Note: `count` vs `for_each` for Conditionals

For simple on/off toggles, `count` is the standard and simplest approach. For conditionally creating resources keyed by name/identity (avoiding index-shift issues), `for_each` with a conditional map/set is often safer:

```hcl
resource "aws_instance" "app" {
  for_each = var.create_instance ? toset(["main"]) : toset([])

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```
