---
name: terraform-patterns
description: Terraform IaCパターン。モジュール設計、状態管理、環境分離、ドリフト検知、インポート
category: DevOps
command: /terraform-patterns
version: 1.0.0
tags: [terraform, iac, infrastructure, modules, state]
---

# Terraform IaC パターンスキル

## 概要

Terraform によるインフラストラクチャ管理のベストプラクティス。
モジュール設計、状態管理、環境分離、ドリフト検知、既存リソースのインポートをカバーする。

## Steps

### Step 1: プロジェクト構成

```
infrastructure/
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── database/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── environments/
│   ├── production/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── development/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       └── backend.tf
└── global/
    ├── iam/
    │   └── main.tf
    └── dns/
        └── main.tf
```

### Step 2: バックエンド設定（状態管理）

```hcl
# environments/production/backend.tf
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
  }

  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "ap-northeast-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
    # state ファイルの暗号化と排他ロック
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "terraform"
      Project     = var.project_name
    }
  }
}
```

**状態管理バックエンドの初期化:**

```hcl
# bootstrap/main.tf - 最初に手動で apply
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state"

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### Step 3: モジュール設計

```hcl
# modules/networking/variables.tf
variable "project_name" {
  type        = string
  description = "プロジェクト名（リソース命名に使用）"
}

variable "environment" {
  type        = string
  description = "環境名"
  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "environment は development, staging, production のいずれか"
  }
}

variable "vpc_cidr" {
  type        = string
  default     = "10.0.0.0/16"
  description = "VPC の CIDR ブロック"
  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "有効な CIDR ブロックを指定してください"
  }
}

variable "availability_zones" {
  type        = list(string)
  description = "使用する AZ のリスト"
}
```

```hcl
# modules/networking/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-${var.environment}-vpc"
  }
}

resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project_name}-${var.environment}-public-${count.index}"
    Type = "public"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 100)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.project_name}-${var.environment}-private-${count.index}"
    Type = "private"
  }
}

resource "aws_nat_gateway" "main" {
  count         = var.environment == "production" ? length(var.availability_zones) : 1
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
}

resource "aws_eip" "nat" {
  count  = var.environment == "production" ? length(var.availability_zones) : 1
  domain = "vpc"
}
```

```hcl
# modules/networking/outputs.tf
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "VPC ID"
}

output "public_subnet_ids" {
  value       = aws_subnet.public[*].id
  description = "パブリックサブネット ID のリスト"
}

output "private_subnet_ids" {
  value       = aws_subnet.private[*].id
  description = "プライベートサブネット ID のリスト"
}
```

### Step 4: 環境別呼び出し

```hcl
# environments/production/main.tf
module "networking" {
  source = "../../modules/networking"

  project_name       = var.project_name
  environment        = "production"
  vpc_cidr           = "10.0.0.0/16"
  availability_zones = ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
}

module "database" {
  source = "../../modules/database"

  project_name      = var.project_name
  environment       = "production"
  vpc_id            = module.networking.vpc_id
  subnet_ids        = module.networking.private_subnet_ids
  instance_class    = "db.r6g.xlarge"
  multi_az          = true
  backup_retention  = 30
  deletion_protection = true
}

module "compute" {
  source = "../../modules/compute"

  project_name   = var.project_name
  environment    = "production"
  vpc_id         = module.networking.vpc_id
  subnet_ids     = module.networking.private_subnet_ids
  instance_type  = "t3.large"
  min_size       = 3
  max_size       = 10
  desired_size   = 3
}
```

### Step 5: ドリフト検知

```hcl
# CI でのドリフト検知（GitHub Actions）
# .github/workflows/terraform-drift.yml
```

```yaml
name: Terraform Drift Detection

on:
  schedule:
    - cron: '0 9 * * 1-5'  # 平日毎朝9時
  workflow_dispatch:

jobs:
  drift:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [production, staging]
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.0

      - name: Terraform Init
        working-directory: environments/${{ matrix.environment }}
        run: terraform init -no-color

      - name: Detect Drift
        id: plan
        working-directory: environments/${{ matrix.environment }}
        run: |
          terraform plan -detailed-exitcode -no-color 2>&1 | tee plan.txt
          echo "exitcode=${PIPESTATUS[0]}" >> "$GITHUB_OUTPUT"
        continue-on-error: true

      - name: Notify on Drift
        if: steps.plan.outputs.exitcode == '2'
        run: |
          # Slack 通知などを送信
          echo "DRIFT DETECTED in ${{ matrix.environment }}"
```

### Step 6: 既存リソースのインポート

```hcl
# Terraform 1.5+ の import ブロック
import {
  to = aws_s3_bucket.existing
  id = "my-existing-bucket"
}

resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"
}

# 複数リソースの一括インポート
import {
  to = aws_security_group.web
  id = "sg-0123456789abcdef0"
}

import {
  to = aws_instance.legacy
  id = "i-0123456789abcdef0"
}
```

**インポート手順:**

```bash
# 1. import ブロックを追加
# 2. plan でインポート対象を確認
terraform plan -generate-config-out=generated.tf

# 3. 生成された設定を確認・調整
# 4. apply でインポート実行
terraform apply

# 5. import ブロックを削除（インポート完了後）
```

### Step 7: セキュリティパターン

```hcl
# sensitive 変数
variable "db_password" {
  type      = string
  sensitive = true
}

# tfsec / checkov による静的解析
# .tfsec.yml
```

```yaml
# GitHub Actions での静的解析
- name: tfsec
  uses: aquasecurity/tfsec-action@v1.0.0
  with:
    working_directory: environments/production

- name: checkov
  uses: bridgecrewio/checkov-action@v12
  with:
    directory: environments/production
    framework: terraform
```

## ベストプラクティス

1. **モジュールは汎用的に**: 環境固有のロジックはモジュール外で制御
2. **変数の validation**: 入力値を早期にバリデーション
3. **prevent_destroy**: 重要リソースには lifecycle で削除防止
4. **default_tags**: プロバイダレベルで共通タグを設定
5. **state の分離**: 環境ごとに state ファイルを分離
6. **ロック**: DynamoDB で同時実行を防止
7. **バージョン固定**: required_version とプロバイダバージョンを固定
8. **plan の確認**: apply 前に必ず plan を確認（CI では -detailed-exitcode を使用）

## よくあるミス

```hcl
# NG: ハードコードされたリージョン
resource "aws_instance" "web" {
  ami = "ami-12345678"  # リージョン依存の AMI ID
}

# OK: data source で動的取得
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_instance" "web" {
  ami = data.aws_ami.amazon_linux.id
}
```

## コマンドリファレンス

```bash
# 初期化
terraform init -upgrade

# フォーマット
terraform fmt -recursive

# バリデーション
terraform validate

# プラン（変更確認）
terraform plan -out=tfplan

# 適用
terraform apply tfplan

# 状態確認
terraform state list
terraform state show aws_instance.web

# リソース移動（リファクタリング時）
terraform state mv aws_instance.old aws_instance.new
```
