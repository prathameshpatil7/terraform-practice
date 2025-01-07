Here's the Terraform project structure to achieve the specified requirements. This includes the creation of a VPC, subnets, S3 bucket for flow logs, flow logs configuration, internet gateway, route tables, and NAT gateways.

---

### Directory Structure
```plaintext
terraform-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
```

---

### `providers.tf`

```bash
provider "aws" {
  region = "us-east-1" # Specify the AWS region
}
```

---

### `variables.tf`

```bash
variable "vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  default = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  default = ["10.0.3.0/24", "10.0.4.0/24"]
}

variable "app_subnet_cidrs" {
  default = ["10.0.5.0/24", "10.0.6.0/24"]
}
```

---

### `main.tf`

```bash
# Create VPC
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "MainVPC"
  }
}

# Create Subnets
resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  map_public_ip_on_launch = true
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = {
    Name = "PublicSubnet-${count.index + 1}"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = {
    Name = "PrivateSubnet-${count.index + 1}"
  }
}

resource "aws_subnet" "app" {
  count             = length(var.app_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.app_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = {
    Name = "AppSubnet-${count.index + 1}"
  }
}

# Create Internet Gateway and Attach to VPC
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "MainIGW"
  }
}

# Create Public Route Table and Associate with Public Subnets
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "PublicRT"
  }
}

resource "aws_route" "public_internet_access" {
  route_table_id         = aws_route_table.public_rt.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}

resource "aws_route_table_association" "public_subnet_assoc" {
  count          = length(var.public_subnet_cidrs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public_rt.id
}

# Create S3 Bucket for Flow Logs
resource "aws_s3_bucket" "flow_logs_bucket" {
  bucket = "vpc-flow-logs-bucket"
  acl    = "private"
  tags = {
    Name = "FlowLogsBucket"
  }
}

# Enable Flow Logs for VPC
resource "aws_flow_log" "vpc_flow_logs" {
  log_destination      = aws_s3_bucket.flow_logs_bucket.arn
  log_destination_type = "s3"
  traffic_type         = "ALL"
  vpc_id               = aws_vpc.main.id
}

# Create NAT Gateways
resource "aws_eip" "nat_gw_eip" {
  count = 2
  vpc   = true
}

resource "aws_nat_gateway" "nat_gw" {
  count         = 2
  allocation_id = aws_eip.nat_gw_eip[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  tags = {
    Name = "NATGW-${count.index + 1}"
  }
}

# Create Route Tables for Private Subnets
resource "aws_route_table" "app_rt" {
  count  = 2
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "AppRT-${count.index + 1}"
  }
}

resource "aws_route" "app_nat_route" {
  count                  = 2
  route_table_id         = aws_route_table.app_rt[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.nat_gw[count.index].id
}

resource "aws_route_table_association" "app_subnet_assoc" {
  count          = length(var.app_subnet_cidrs)
  subnet_id      = aws_subnet.app[count.index].id
  route_table_id = element(aws_route_table.app_rt[*].id, count.index % 2)
}
```

---

### `outputs.tf`

```bash
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

output "app_subnet_ids" {
  value = aws_subnet.app[*].id
}

output "internet_gateway_id" {
  value = aws_internet_gateway.igw.id
}

output "nat_gateway_ids" {
  value = aws_nat_gateway.nat_gw[*].id
}
```

---

### How to Use

1. **Initialize the Terraform Project**:
   ```bash
   terraform init
   ```

2. **Preview the Changes**:
   ```bash
   terraform plan
   ```

3. **Apply the Configuration**:
   ```bash
   terraform apply
   ```

Let me know if you need further assistance!
