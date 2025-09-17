# Terraform環境でのOS移行戦略

## 目次

1. [Terraform構成の現状分析](#1-terraform構成の現状分析)
2. [OS移行戦略](#2-os移行戦略)
3. [段階的移行実装](#3-段階的移行実装)
4. [Blue-Green移行パターン](#4-blue-green移行パターン)
5. [実装例](#5-実装例)

---

## 1. Terraform構成の現状分析

### 1.1 典型的なディレクトリ構成

```
project/
├── infra/
│   ├── modules/              # 再利用可能なモジュール
│   │   ├── ecs/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   ├── ec2/
│   │   ├── rds/
│   │   ├── alb/
│   │   └── vpc/
│   └── envs/                 # 環境固有設定
│       ├── dev/
│       │   ├── main.tf
│       │   ├── variables.tf
│       │   └── terraform.tfvars
│       ├── staging/
│       └── prod/
```

### 1.2 OS移行における課題

1. **AMI変更の影響範囲**
   - ECS/EC2インスタンスの再作成が必要
   - ダウンタイムの発生
   - データ永続化の考慮

2. **環境間の整合性**
   - dev → staging → prod の順次移行
   - 設定値の統一管理

3. **ロールバック戦略**
   - 問題発生時の迅速な復旧
   - 状態ファイルの管理

---

## 2. OS移行戦略

### 2.1 移行パターンの選択

#### パターンA: Blue-Green移行（推奨）
- 新環境を並行構築
- トラフィック切り替えによる移行
- 最小ダウンタイム

#### パターンB: ローリング移行
- インスタンス単位での段階的移行
- サービス単位での個別移行
- 長期間の混在環境

#### パターンC: 全体再構築
- 全リソースの一括更新
- 計画的メンテナンス窓での実行
- シンプルな実装

### 2.2 移行戦略決定フローチャート

```
OS移行戦略決定
│
├─ ダウンタイム許容度は？
│  ├─ 最小限 → Blue-Green移行
│  ├─ 数時間OK → ローリング移行
│  └─ 半日OK → 全体再構築
│
├─ 環境規模は？
│  ├─ 大規模 → Blue-Green移行
│  ├─ 中規模 → ローリング移行
│  └─ 小規模 → 全体再構築
│
└─ 運用コストは？
   ├─ 低コスト重視 → 全体再構築
   ├─ バランス重視 → ローリング移行
   └─ 安全性重視 → Blue-Green移行
```

---

## 3. 段階的移行実装

### 3.1 モジュール設計パターン

#### 3.1.1 OS対応モジュール構造

```hcl
# modules/compute/variables.tf
variable "os_type" {
  description = "Operating system type"
  type        = string
  validation {
    condition = contains(["amazon-linux-2", "almalinux-9", "amazon-linux-2023"], var.os_type)
    error_message = "Supported OS types: amazon-linux-2, almalinux-9, amazon-linux-2023"
  }
}

variable "migration_stage" {
  description = "Migration stage"
  type        = string
  default     = "current"
  validation {
    condition = contains(["current", "migration", "new"], var.migration_stage)
    error_message = "Valid stages: current, migration, new"
  }
}
```

#### 3.1.2 AMI データソース管理

```hcl
# modules/compute/data.tf
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

data "aws_ami" "almalinux_9" {
  most_recent = true
  owners      = ["679593333241"]  # AlmaLinux OS Foundation
  filter {
    name   = "name"
    values = ["AlmaLinux-9-*"]
  }
  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
}

data "aws_ami" "amazon_linux_2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*"]
  }
}

locals {
  ami_map = {
    "amazon-linux-2"    = data.aws_ami.amazon_linux_2.id
    "almalinux-9"        = data.aws_ami.almalinux_9.id
    "amazon-linux-2023" = data.aws_ami.amazon_linux_2023.id
  }
}
```

### 3.2 環境設定管理

#### 3.2.1 terraform.tfvarsでの OS管理

```hcl
# envs/dev/terraform.tfvars
os_type = "amazon-linux-2"
migration_stage = "current"

# 移行時の設定例
# envs/dev/terraform.tfvars (移行中)
os_type = "almalinux-9"
migration_stage = "migration"
enable_blue_green = true
```

#### 3.2.2 段階的変数管理

```hcl
# envs/dev/variables.tf
variable "migration_config" {
  description = "Migration configuration"
  type = object({
    current_os    = string
    target_os     = string
    migration_percentage = number
    enable_blue_green    = bool
  })
  default = {
    current_os    = "amazon-linux-2"
    target_os     = "almalinux-9"
    migration_percentage = 0
    enable_blue_green    = false
  }
}
```

---

## 4. Blue-Green移行パターン

### 4.1 Blue-Green環境の設計

#### 4.1.1 ベースモジュール構造

```hcl
# modules/blue_green_deployment/main.tf
resource "aws_launch_template" "blue" {
  count = var.enable_blue_environment ? 1 : 0
  
  name_prefix   = "${var.service_name}-blue-"
  image_id      = local.ami_map[var.current_os]
  instance_type = var.instance_type
  
  vpc_security_group_ids = var.security_group_ids
  
  user_data = base64encode(templatefile("${path.module}/userdata/${var.current_os}.sh", {
    service_name = var.service_name
  }))
  
  tag_specifications {
    resource_type = "instance"
    tags = merge(var.common_tags, {
      Environment = "blue"
      OS          = var.current_os
    })
  }
}

resource "aws_launch_template" "green" {
  count = var.enable_green_environment ? 1 : 0
  
  name_prefix   = "${var.service_name}-green-"
  image_id      = local.ami_map[var.target_os]
  instance_type = var.instance_type
  
  vpc_security_group_ids = var.security_group_ids
  
  user_data = base64encode(templatefile("${path.module}/userdata/${var.target_os}.sh", {
    service_name = var.service_name
  }))
  
  tag_specifications {
    resource_type = "instance"
    tags = merge(var.common_tags, {
      Environment = "green"
      OS          = var.target_os
    })
  }
}
```

#### 4.1.2 Auto Scaling Group管理

```hcl
# Blue環境ASG
resource "aws_autoscaling_group" "blue" {
  count = var.enable_blue_environment ? 1 : 0
  
  name                = "${var.service_name}-blue-asg"
  vpc_zone_identifier = var.subnet_ids
  target_group_arns   = [aws_lb_target_group.blue[0].arn]
  health_check_type   = "ELB"
  
  min_size         = var.blue_min_size
  max_size         = var.blue_max_size
  desired_capacity = var.blue_desired_capacity
  
  launch_template {
    id      = aws_launch_template.blue[0].id
    version = "$Latest"
  }
  
  tag {
    key                 = "Name"
    value               = "${var.service_name}-blue"
    propagate_at_launch = true
  }
}

# Green環境ASG
resource "aws_autoscaling_group" "green" {
  count = var.enable_green_environment ? 1 : 0
  
  name                = "${var.service_name}-green-asg"
  vpc_zone_identifier = var.subnet_ids
  target_group_arns   = [aws_lb_target_group.green[0].arn]
  health_check_type   = "ELB"
  
  min_size         = var.green_min_size
  max_size         = var.green_max_size
  desired_capacity = var.green_desired_capacity
  
  launch_template {
    id      = aws_launch_template.green[0].id
    version = "$Latest"
  }
  
  tag {
    key                 = "Name"
    value               = "${var.service_name}-green"
    propagate_at_launch = true
  }
}
```

#### 4.1.3 ロードバランサー設定

```hcl
# Blue Target Group
resource "aws_lb_target_group" "blue" {
  count = var.enable_blue_environment ? 1 : 0
  
  name     = "${var.service_name}-blue-tg"
  port     = var.target_port
  protocol = "HTTP"
  vpc_id   = var.vpc_id
  
  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = var.health_check_path
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }
  
  tags = merge(var.common_tags, {
    Environment = "blue"
  })
}

# Green Target Group
resource "aws_lb_target_group" "green" {
  count = var.enable_green_environment ? 1 : 0
  
  name     = "${var.service_name}-green-tg"
  port     = var.target_port
  protocol = "HTTP"
  vpc_id   = var.vpc_id
  
  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = var.health_check_path
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }
  
  tags = merge(var.common_tags, {
    Environment = "green"
  })
}

# ALB Listener Rule（重み付けルーティング）
resource "aws_lb_listener_rule" "weighted_routing" {
  listener_arn = var.listener_arn
  priority     = var.rule_priority
  
  action {
    type = "forward"
    forward {
      dynamic "target_group" {
        for_each = var.enable_blue_environment ? [1] : []
        content {
          arn    = aws_lb_target_group.blue[0].arn
          weight = var.blue_weight
        }
      }
      
      dynamic "target_group" {
        for_each = var.enable_green_environment ? [1] : []
        content {
          arn    = aws_lb_target_group.green[0].arn
          weight = var.green_weight
        }
      }
    }
  }
  
  condition {
    path_pattern {
      values = [var.path_pattern]
    }
  }
}
```

### 4.2 ECS環境のBlue-Green移行

#### 4.2.1 ECS Blue-Green Service

```hcl
# modules/ecs_blue_green/main.tf
resource "aws_ecs_service" "blue" {
  count = var.enable_blue_environment ? 1 : 0
  
  name            = "${var.service_name}-blue"
  cluster         = var.cluster_id
  task_definition = aws_ecs_task_definition.blue[0].arn
  desired_count   = var.blue_desired_count
  
  load_balancer {
    target_group_arn = var.blue_target_group_arn
    container_name   = var.container_name
    container_port   = var.container_port
  }
  
  tags = merge(var.common_tags, {
    Environment = "blue"
    OS          = var.current_os
  })
}

resource "aws_ecs_service" "green" {
  count = var.enable_green_environment ? 1 : 0
  
  name            = "${var.service_name}-green"
  cluster         = var.cluster_id
  task_definition = aws_ecs_task_definition.green[0].arn
  desired_count   = var.green_desired_count
  
  load_balancer {
    target_group_arn = var.green_target_group_arn
    container_name   = var.container_name
    container_port   = var.container_port
  }
  
  tags = merge(var.common_tags, {
    Environment = "green"
    OS          = var.target_os
  })
}
```

#### 4.2.2 ECS Task Definition

```hcl
# Task Definition for Blue (current OS)
resource "aws_ecs_task_definition" "blue" {
  count = var.enable_blue_environment ? 1 : 0
  
  family                   = "${var.service_name}-blue"
  requires_compatibilities = ["EC2"]
  network_mode            = "bridge"
  
  container_definitions = jsonencode([
    {
      name  = var.container_name
      image = "${var.image_repository}:${var.current_image_tag}"
      
      portMappings = [
        {
          containerPort = var.container_port
          protocol      = "tcp"
        }
      ]
      
      environment = concat(var.common_environment, [
        {
          name  = "OS_VERSION"
          value = var.current_os
        }
      ])
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.blue[0].name
          "awslogs-region"        = data.aws_region.current.name
          "awslogs-stream-prefix" = "ecs"
        }
      }
    }
  ])
}

# Task Definition for Green (new OS)
resource "aws_ecs_task_definition" "green" {
  count = var.enable_green_environment ? 1 : 0
  
  family                   = "${var.service_name}-green"
  requires_compatibilities = ["EC2"]
  network_mode            = "bridge"
  
  container_definitions = jsonencode([
    {
      name  = var.container_name
      image = "${var.image_repository}:${var.target_image_tag}"
      
      portMappings = [
        {
          containerPort = var.container_port
          protocol      = "tcp"
        }
      ]
      
      environment = concat(var.common_environment, [
        {
          name  = "OS_VERSION"
          value = var.target_os
        }
      ])
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.green[0].name
          "awslogs-region"        = data.aws_region.current.name
          "awslogs-stream-prefix" = "ecs"
        }
      }
    }
  ])
}
```

---

## 5. 実装例

### 5.1 ディレクトリ構成例

```
infra/
├── modules/
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── userdata/
│   │       ├── amazon-linux-2.sh
│   │       ├── almalinux-9.sh
│   │       └── amazon-linux-2023.sh
│   ├── ecs/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── blue_green_deployment/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── ecs_blue_green/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── envs/
    ├── dev/
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── terraform.tfvars
    │   └── migration/
    │       ├── migration.tfvars
    │       └── rollback.tfvars
    ├── staging/
    └── prod/
```

### 5.2 環境別実装

#### 5.2.1 開発環境（dev）

```hcl
# envs/dev/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket = "your-terraform-state-bucket"
    key    = "dev/terraform.tfstate"
    region = "us-west-2"
  }
}

provider "aws" {
  region = var.aws_region
}

# Blue-Green Deployment Module
module "web_service_blue_green" {
  source = "../../modules/blue_green_deployment"
  
  service_name = "web-service"
  
  # OS Configuration
  current_os = var.migration_config.current_os
  target_os  = var.migration_config.target_os
  
  # Blue Environment
  enable_blue_environment = var.migration_config.enable_blue_environment
  blue_min_size          = var.migration_config.blue_min_size
  blue_max_size          = var.migration_config.blue_max_size
  blue_desired_capacity  = var.migration_config.blue_desired_capacity
  blue_weight           = var.migration_config.blue_weight
  
  # Green Environment
  enable_green_environment = var.migration_config.enable_green_environment
  green_min_size          = var.migration_config.green_min_size
  green_max_size          = var.migration_config.green_max_size
  green_desired_capacity  = var.migration_config.green_desired_capacity
  green_weight           = var.migration_config.green_weight
  
  # Network Configuration
  vpc_id         = data.aws_vpc.main.id
  subnet_ids     = data.aws_subnets.private.ids
  security_group_ids = [aws_security_group.web.id]
  
  # Load Balancer
  listener_arn = aws_lb_listener.web.arn
  rule_priority = 100
  
  common_tags = var.common_tags
}

# ECS Blue-Green Module
module "api_service_ecs_blue_green" {
  source = "../../modules/ecs_blue_green"
  
  service_name = "api-service"
  cluster_id   = aws_ecs_cluster.main.id
  
  # Container Configuration
  container_name     = "api"
  container_port     = 8080
  image_repository   = "your-ecr-repo/api"
  current_image_tag  = var.current_image_tag
  target_image_tag   = var.target_image_tag
  
  # OS Configuration
  current_os = var.migration_config.current_os
  target_os  = var.migration_config.target_os
  
  # Environment Configuration
  enable_blue_environment  = var.migration_config.enable_blue_environment
  enable_green_environment = var.migration_config.enable_green_environment
  blue_desired_count      = var.migration_config.blue_desired_count
  green_desired_count     = var.migration_config.green_desired_count
  
  # Target Groups
  blue_target_group_arn  = module.web_service_blue_green.blue_target_group_arn
  green_target_group_arn = module.web_service_blue_green.green_target_group_arn
  
  common_environment = [
    {
      name  = "ENVIRONMENT"
      value = "dev"
    },
    {
      name  = "LOG_LEVEL"
      value = "debug"
    }
  ]
  
  common_tags = var.common_tags
}
```

#### 5.2.2 開発環境変数設定

```hcl
# envs/dev/variables.tf
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}

variable "migration_config" {
  description = "Migration configuration"
  type = object({
    current_os = string
    target_os  = string
    
    enable_blue_environment  = bool
    enable_green_environment = bool
    
    blue_min_size         = number
    blue_max_size         = number
    blue_desired_capacity = number
    blue_weight          = number
    
    green_min_size         = number
    green_max_size         = number
    green_desired_capacity = number
    green_weight          = number
    
    blue_desired_count  = number
    green_desired_count = number
  })
}

variable "current_image_tag" {
  description = "Current image tag for blue environment"
  type        = string
  default     = "v1.0.0"
}

variable "target_image_tag" {
  description = "Target image tag for green environment"
  type        = string
  default     = "v1.1.0"
}

variable "common_tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default = {
    Environment = "dev"
    Project     = "os-migration"
    ManagedBy   = "terraform"
  }
}
```

#### 5.2.3 段階的移行設定ファイル

```hcl
# envs/dev/terraform.tfvars （現在の環境）
migration_config = {
  current_os = "amazon-linux-2"
  target_os  = "almalinux-9"
  
  enable_blue_environment  = true
  enable_green_environment = false
  
  blue_min_size         = 2
  blue_max_size         = 6
  blue_desired_capacity = 3
  blue_weight          = 100
  
  green_min_size         = 0
  green_max_size         = 0
  green_desired_capacity = 0
  green_weight          = 0
  
  blue_desired_count  = 2
  green_desired_count = 0
}
```

```hcl
# envs/dev/migration/phase1.tfvars （Green環境追加）
migration_config = {
  current_os = "amazon-linux-2"
  target_os  = "almalinux-9"
  
  enable_blue_environment  = true
  enable_green_environment = true
  
  blue_min_size         = 2
  blue_max_size         = 6
  blue_desired_capacity = 3
  blue_weight          = 90
  
  green_min_size         = 1
  green_max_size         = 3
  green_desired_capacity = 1
  green_weight          = 10
  
  blue_desired_count  = 2
  green_desired_count = 1
}
```

```hcl
# envs/dev/migration/phase2.tfvars （50:50分散）
migration_config = {
  current_os = "amazon-linux-2"
  target_os  = "almalinux-9"
  
  enable_blue_environment  = true
  enable_green_environment = true
  
  blue_min_size         = 2
  blue_max_size         = 4
  blue_desired_capacity = 2
  blue_weight          = 50
  
  green_min_size         = 2
  green_max_size         = 4
  green_desired_capacity = 2
  green_weight          = 50
  
  blue_desired_count  = 1
  green_desired_count = 2
}
```

```hcl
# envs/dev/migration/phase3.tfvars （Green完全移行）
migration_config = {
  current_os = "amazon-linux-2"
  target_os  = "almalinux-9"
  
  enable_blue_environment  = false
  enable_green_environment = true
  
  blue_min_size         = 0
  blue_max_size         = 0
  blue_desired_capacity = 0
  blue_weight          = 0
  
  green_min_size         = 2
  green_max_size         = 6
  green_desired_capacity = 3
  green_weight          = 100
  
  blue_desired_count  = 0
  green_desired_count = 3
}
```

### 5.3 移行実行コマンド

#### 5.3.1 段階的移行コマンド

```bash
#!/bin/bash
# scripts/migrate.sh

set -e

ENVIRONMENT=$1
PHASE=$2

if [ -z "$ENVIRONMENT" ] || [ -z "$PHASE" ]; then
    echo "Usage: $0 <environment> <phase>"
    echo "Example: $0 dev phase1"
    exit 1
fi

cd "infra/envs/$ENVIRONMENT"

echo "=== OS Migration Phase: $PHASE ==="
echo "Environment: $ENVIRONMENT"
echo "Phase: $PHASE"

# Terraform plan with specific phase configuration
echo "=== Terraform Plan ==="
terraform plan -var-file="migration/${PHASE}.tfvars" -out="migration-${PHASE}.tfplan"

echo "=== Plan Review ==="
echo "Please review the above plan. Continue? (y/N)"
read -r confirm

if [ "$confirm" != "y" ] && [ "$confirm" != "Y" ]; then
    echo "Migration cancelled."
    exit 0
fi

echo "=== Terraform Apply ==="
terraform apply "migration-${PHASE}.tfplan"

echo "=== Migration Phase $PHASE Completed ==="

# ヘルスチェック
echo "=== Health Check ==="
sleep 30
./scripts/health_check.sh "$ENVIRONMENT"

echo "=== Migration Phase $PHASE Successfully Completed ==="
```

#### 5.3.2 ロールバックスクリプト

```bash
#!/bin/bash
# scripts/rollback.sh

set -e

ENVIRONMENT=$1

if [ -z "$ENVIRONMENT" ]; then
    echo "Usage: $0 <environment>"
    exit 1
fi

cd "infra/envs/$ENVIRONMENT"

echo "=== Emergency Rollback ==="
echo "Environment: $ENVIRONMENT"

# Blue環境にトラフィック100%戻す
echo "=== Rolling back to Blue environment ==="
terraform apply -var-file="migration/rollback.tfvars" -auto-approve

echo "=== Rollback completed ==="
```

#### 5.3.3 ヘルスチェックスクリプト

```bash
#!/bin/bash
# scripts/health_check.sh

ENVIRONMENT=$1
ENDPOINT="https://api-${ENVIRONMENT}.yourdomain.com"

echo "=== Health Check for $ENVIRONMENT ==="

# API Health Check
echo "Checking API health..."
response=$(curl -s -o /dev/null -w "%{http_code}" "$ENDPOINT/health")

if [ "$response" -eq 200 ]; then
    echo "✅ API health check passed"
else
    echo "❌ API health check failed (HTTP $response)"
    exit 1
fi

# Infrastructure Health Check
echo "Checking infrastructure..."
cd "infra/envs/$ENVIRONMENT"

# Target Group Health
aws elbv2 describe-target-health \
    --target-group-arn $(terraform output -raw blue_target_group_arn) \
    --query 'TargetHealthDescriptions[?TargetHealth.State!=`healthy`]' \
    --output table

aws elbv2 describe-target-health \
    --target-group-arn $(terraform output -raw green_target_group_arn) \
    --query 'TargetHealthDescriptions[?TargetHealth.State!=`healthy`]' \
    --output table

echo "=== Health Check Completed ==="
```

### 5.4 CI/CD統合

#### 5.4.1 GitHub Actions ワークフロー

```yaml
# .github/workflows/os-migration.yml
name: OS Migration

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
        - dev
        - staging
        - prod
      phase:
        description: 'Migration phase'
        required: true
        type: choice
        options:
        - phase1
        - phase2
        - phase3
        - rollback

jobs:
  migrate:
    name: Execute Migration
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
      
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: 1.5.0
        
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2
        
    - name: Terraform Init
      working-directory: infra/envs/${{ github.event.inputs.environment }}
      run: terraform init
      
    - name: Terraform Plan
      working-directory: infra/envs/${{ github.event.inputs.environment }}
      run: |
        terraform plan \
          -var-file="migration/${{ github.event.inputs.phase }}.tfvars" \
          -out="migration-${{ github.event.inputs.phase }}.tfplan"
          
    - name: Terraform Apply
      working-directory: infra/envs/${{ github.event.inputs.environment }}
      run: terraform apply "migration-${{ github.event.inputs.phase }}.tfplan"
      
    - name: Health Check
      run: ./scripts/health_check.sh ${{ github.event.inputs.environment }}
      
    - name: Notify Slack
      if: always()
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        channel: '#ops'
        text: |
          OS Migration ${{ github.event.inputs.phase }} for ${{ github.event.inputs.environment }}
          Status: ${{ job.status }}
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## まとめ

このTerraform環境でのOS移行戦略では、以下の要素を組み合わせています：

1. **モジュール化設計**: 再利用可能なモジュールでOS移行を管理
2. **Blue-Green移行**: 最小ダウンタイムでの安全な移行
3. **段階的実行**: フェーズ別の設定ファイルで管理
4. **自動化**: CI/CDパイプラインとの統合
5. **監視**: ヘルスチェックとロールバック機能

この構成により、インフラストラクチャをコードとして管理しながら、安全で確実なOS移行が実現できます。