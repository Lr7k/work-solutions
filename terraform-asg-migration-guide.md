# Terraform管理下でのASG起動設定から起動テンプレートへの移行手順

## 概要
`infra/module`と`infra/env`の構成でTerraformを管理している環境での、Auto Scaling Groupの起動設定から起動テンプレートへの移行手順です。

## 前提条件
- Terraformディレクトリ構成: `infra/module/`（共通モジュール）、`infra/env/`（環境固有設定）
- 適切なTerraform権限とAWS権限
- 既存のASG設定がTerraformで管理されている

## ディレクトリ構成例
```
project/
├── infra/
│   ├── module/
│   │   ├── asg/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   ├── launch_template/      # 新規作成
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   └── ...
│   └── env/
│       ├── dev/
│       │   ├── main.tf
│       │   ├── variables.tf
│       │   └── terraform.tfvars
│       ├── staging/
│       └── prod/
```

## 1. 現状のTerraform構成確認

### 1.1 既存のASGモジュール確認
```bash
# 既存のASG設定を確認
find infra/module -name "*.tf" -exec grep -l "aws_autoscaling_group\|aws_launch_configuration" {} \;

# 現在の設定内容を確認
cat infra/module/asg/main.tf
```

### 1.2 環境固有設定の確認
```bash
# 各環境での設定を確認
ls infra/env/*/
cat infra/env/dev/terraform.tfvars
```

## 2. 起動テンプレートモジュールの作成

### 2.1 launch_templateモジュールの作成
`infra/module/launch_template/main.tf`:
```hcl
# Launch Template
resource "aws_launch_template" "this" {
  name_prefix   = var.name_prefix
  description   = var.description
  
  image_id      = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name
  
  vpc_security_group_ids = var.security_group_ids
  
  dynamic "iam_instance_profile" {
    for_each = var.iam_instance_profile_name != "" ? [1] : []
    content {
      name = var.iam_instance_profile_name
    }
  }
  
  user_data = var.user_data_base64
  
  dynamic "block_device_mappings" {
    for_each = var.block_device_mappings
    content {
      device_name = block_device_mappings.value.device_name
      ebs {
        volume_size           = block_device_mappings.value.volume_size
        volume_type           = block_device_mappings.value.volume_type
        delete_on_termination = block_device_mappings.value.delete_on_termination
        encrypted             = block_device_mappings.value.encrypted
      }
    }
  }
  
  monitoring {
    enabled = var.enable_monitoring
  }
  
  dynamic "tag_specifications" {
    for_each = var.tag_specifications
    content {
      resource_type = tag_specifications.value.resource_type
      tags          = tag_specifications.value.tags
    }
  }
  
  tags = var.tags
  
  lifecycle {
    create_before_destroy = true
  }
}
```

### 2.2 launch_templateモジュールの変数定義
`infra/module/launch_template/variables.tf`:
```hcl
variable "name_prefix" {
  description = "Launch template name prefix"
  type        = string
}

variable "description" {
  description = "Launch template description"
  type        = string
  default     = ""
}

variable "ami_id" {
  description = "AMI ID for the launch template"
  type        = string
}

variable "instance_type" {
  description = "Instance type"
  type        = string
}

variable "key_name" {
  description = "Key pair name"
  type        = string
  default     = ""
}

variable "security_group_ids" {
  description = "Security group IDs"
  type        = list(string)
}

variable "iam_instance_profile_name" {
  description = "IAM instance profile name"
  type        = string
  default     = ""
}

variable "user_data_base64" {
  description = "User data in base64 encoded format"
  type        = string
  default     = ""
}

variable "block_device_mappings" {
  description = "Block device mappings"
  type = list(object({
    device_name           = string
    volume_size          = number
    volume_type          = string
    delete_on_termination = bool
    encrypted            = bool
  }))
  default = []
}

variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = true
}

variable "tag_specifications" {
  description = "Tag specifications for resources created by instances"
  type = list(object({
    resource_type = string
    tags          = map(string)
  }))
  default = []
}

variable "tags" {
  description = "Tags for the launch template"
  type        = map(string)
  default     = {}
}
```

### 2.3 launch_templateモジュールのアウトプット
`infra/module/launch_template/outputs.tf`:
```hcl
output "launch_template_id" {
  description = "Launch template ID"
  value       = aws_launch_template.this.id
}

output "launch_template_name" {
  description = "Launch template name"
  value       = aws_launch_template.this.name
}

output "launch_template_latest_version" {
  description = "Launch template latest version"
  value       = aws_launch_template.this.latest_version
}

output "launch_template_arn" {
  description = "Launch template ARN"
  value       = aws_launch_template.this.arn
}
```

## 3. ASGモジュールの更新

### 3.1 既存ASGモジュールの修正
`infra/module/asg/main.tf`を更新:
```hcl
# Auto Scaling Group (Launch Template版)
resource "aws_autoscaling_group" "this" {
  name                = var.name
  vpc_zone_identifier = var.subnet_ids
  target_group_arns   = var.target_group_arns
  health_check_type   = var.health_check_type
  health_check_grace_period = var.health_check_grace_period
  
  min_size         = var.min_size
  max_size         = var.max_size
  desired_capacity = var.desired_capacity
  
  # Launch Template使用（新しい設定）
  dynamic "launch_template" {
    for_each = var.launch_template_id != "" ? [1] : []
    content {
      id      = var.launch_template_id
      version = var.launch_template_version
    }
  }
  
  # Launch Configuration使用（既存設定、互換性のため残す）
  launch_configuration = var.launch_configuration_name != "" ? var.launch_configuration_name : null
  
  dynamic "tag" {
    for_each = var.tags
    content {
      key                 = tag.key
      value               = tag.value
      propagate_at_launch = true
    }
  }
  
  lifecycle {
    create_before_destroy = true
    ignore_changes       = [desired_capacity]
  }
}

# 既存のLaunch Configuration（段階的移行のため残す）
resource "aws_launch_configuration" "this" {
  count = var.launch_configuration_name != "" && var.launch_template_id == "" ? 1 : 0
  
  name_prefix          = var.launch_configuration_name_prefix
  image_id             = var.ami_id
  instance_type        = var.instance_type
  key_name             = var.key_name
  security_groups      = var.security_group_ids
  iam_instance_profile = var.iam_instance_profile_name
  user_data           = var.user_data_base64
  
  enable_monitoring = var.enable_monitoring
  
  dynamic "ebs_block_device" {
    for_each = var.ebs_block_devices
    content {
      device_name           = ebs_block_device.value.device_name
      volume_size          = ebs_block_device.value.volume_size
      volume_type          = ebs_block_device.value.volume_type
      delete_on_termination = ebs_block_device.value.delete_on_termination
      encrypted            = ebs_block_device.value.encrypted
    }
  }
  
  lifecycle {
    create_before_destroy = true
  }
}
```

### 3.2 ASGモジュールの変数更新
`infra/module/asg/variables.tf`に追加:
```hcl
# Launch Template関連の変数を追加
variable "launch_template_id" {
  description = "Launch template ID (use this OR launch_configuration_name)"
  type        = string
  default     = ""
}

variable "launch_template_version" {
  description = "Launch template version"
  type        = string
  default     = "$Latest"
}

# Launch Configuration関連（既存、互換性のため）
variable "launch_configuration_name" {
  description = "Launch configuration name (use this OR launch_template_id)"
  type        = string
  default     = ""
}

variable "launch_configuration_name_prefix" {
  description = "Launch configuration name prefix"
  type        = string
  default     = ""
}

# 共通の変数（既存のものをそのまま利用）
variable "ami_id" {
  description = "AMI ID"
  type        = string
}

variable "instance_type" {
  description = "Instance type"
  type        = string
}

# ... その他の既存変数
```

## 4. 環境固有設定の更新

### 4.1 dev環境の更新例
`infra/env/dev/main.tf`:
```hcl
# Launch Template モジュール（新規追加）
module "launch_template" {
  source = "../../module/launch_template"
  
  name_prefix    = var.launch_template_name_prefix
  description    = var.launch_template_description
  ami_id         = var.ami_id
  instance_type  = var.instance_type
  key_name       = var.key_name
  security_group_ids = [module.security_group.security_group_id]
  iam_instance_profile_name = var.iam_instance_profile_name
  user_data_base64 = base64encode(var.user_data)
  
  block_device_mappings = var.block_device_mappings
  enable_monitoring     = var.enable_monitoring
  
  tag_specifications = [
    {
      resource_type = "instance"
      tags = merge(var.default_tags, {
        Name = "${var.project_name}-${var.environment}-instance"
      })
    }
  ]
  
  tags = merge(var.default_tags, {
    Name = "${var.project_name}-${var.environment}-launch-template"
  })
}

# ASG モジュール（更新）
module "asg" {
  source = "../../module/asg"
  
  name            = "${var.project_name}-${var.environment}-asg"
  subnet_ids      = var.private_subnet_ids
  target_group_arns = [module.alb.target_group_arn]
  
  min_size         = var.asg_min_size
  max_size         = var.asg_max_size
  desired_capacity = var.asg_desired_capacity
  
  # Launch Templateを使用（新しい設定）
  launch_template_id      = module.launch_template.launch_template_id
  launch_template_version = "$Latest"
  
  # Launch Configurationは使用しない
  # launch_configuration_name = ""
  
  health_check_type         = var.health_check_type
  health_check_grace_period = var.health_check_grace_period
  
  tags = var.default_tags
}
```

### 4.2 terraform.tfvarsの更新
`infra/env/dev/terraform.tfvars`:
```hcl
# 既存の設定はそのまま
project_name = "my-project"
environment  = "dev"

# Launch Template用の新しい設定
launch_template_name_prefix = "my-project-dev-template"
launch_template_description = "Launch template for dev environment"

# 既存設定をそのまま利用
ami_id               = "ami-xxxxxxxxx"
instance_type        = "t3.micro"
key_name            = "my-dev-key"
iam_instance_profile_name = "my-dev-instance-profile"

user_data = <<-EOF
#!/bin/bash
yum update -y
yum install -y docker
systemctl start docker
systemctl enable docker
EOF

block_device_mappings = [
  {
    device_name           = "/dev/xvda"
    volume_size          = 20
    volume_type          = "gp3"
    delete_on_termination = true
    encrypted            = true
  }
]

enable_monitoring = true

# ASG設定
asg_min_size         = 1
asg_max_size         = 3
asg_desired_capacity = 2

health_check_type         = "ELB"
health_check_grace_period = 300

default_tags = {
  Project     = "my-project"
  Environment = "dev"
  Terraform   = "true"
}
```

## 5. 移行実行手順

### 5.1 段階的移行の実行
```bash
# 1. 現在の状態をバックアップ
cd infra/env/dev
terraform plan -out=current.tfplan

# 2. Launch Template作成のみ実行
terraform apply -target=module.launch_template

# 3. ASGの更新（Launch Templateに切り替え）
terraform plan  # 変更内容を確認
terraform apply

# 4. 動作確認後、不要なLaunch Configurationを削除
# （必要に応じて、ASGモジュールからLaunch Configuration部分を削除）
```

### 5.2 全環境への適用
```bash
# staging環境
cd ../staging
terraform plan
terraform apply

# prod環境（最も慎重に）
cd ../prod
terraform plan -out=prod.tfplan
# プランを十分に確認してから実行
terraform apply prod.tfplan
```

## 6. 検証とロールバック

### 6.1 検証手順
```bash
# Terraform状態の確認
terraform show | grep -A 10 "aws_autoscaling_group"

# AWS CLIでの確認
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $(terraform output asg_name)

# インスタンス起動テスト
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name $(terraform output asg_name) \
  --desired-capacity $((current_capacity + 1))
```

### 6.2 ロールバック手順
```bash
# 緊急時：Launch Configurationに戻す
# main.tfでlaunch_template_idをコメントアウトし、launch_configuration_nameを有効化
terraform plan
terraform apply

# 完全ロールバック
terraform plan -destroy -target=module.launch_template
terraform apply -destroy -target=module.launch_template
```

## 7. 最適化と運用

### 7.1 Mixed Instance Policy対応
ASGモジュールにMixed Instance Policyを追加:
```hcl
dynamic "mixed_instances_policy" {
  for_each = var.mixed_instances_policy_enabled ? [1] : []
  content {
    launch_template {
      launch_template_specification {
        launch_template_id = var.launch_template_id
        version           = var.launch_template_version
      }
      
      dynamic "override" {
        for_each = var.instance_type_overrides
        content {
          instance_type     = override.value.instance_type
          weighted_capacity = override.value.weighted_capacity
        }
      }
    }
    
    instances_distribution {
      on_demand_percentage = var.on_demand_percentage
      spot_allocation_strategy = var.spot_allocation_strategy
    }
  }
}
```

### 7.2 バージョン管理
Launch Templateのバージョン管理戦略：
- 開発環境: `$Latest`
- 本番環境: 特定のバージョン番号

## チェックリスト

- [ ] 既存のTerraform構成確認完了
- [ ] Launch Templateモジュール作成完了
- [ ] ASGモジュール更新完了
- [ ] 環境固有設定更新完了
- [ ] dev環境での検証完了
- [ ] staging環境での検証完了
- [ ] prod環境での移行完了
- [ ] 不要なLaunch Configuration削除完了
- [ ] 文書化・手順書更新完了