# ASG 起動テンプレート移行 & OS イメージ更新手順書

## 概要
本手順書では、Terraform で管理している AWS インフラに対して以下の作業を実施します：
1. Auto Scaling Group (ASG) の起動方法を起動設定から起動テンプレートへ変更
2. OS イメージ（AMI）の更新

## 前提条件
- Terraform コードは GitLab で管理されている
- GitLab Pipeline を通じて `terraform plan` および `terraform apply` を実行可能
- 環境: intg, stg, prod の順に作業を実施
- stg, prod は intg での動作確認完了後に実施

## 重要な注意事項
⚠️ **Terraform apply 後、既存インスタンスは自動的に置き換わりません**
- Terraform は ASG の設定（起動テンプレート、AMI ID）のみを更新
- 既存インスタンスを新しい設定で入れ替えるには、AWS マネジメントコンソールまたは CLI でインスタンスリフレッシュが必要

---

## 作業フェーズ

### フェーズ 1: Terraform コード変更（全環境共通）

#### 1.1 起動テンプレートの作成
既存の起動設定（Launch Configuration）の設定を元に、起動テンプレート（Launch Template）を定義します。

```hcl
# 例: main.tf または launch_template.tf
resource "aws_launch_template" "example" {
  name_prefix   = "example-lt-"
  image_id      = var.ami_id  # 新しい AMI ID を指定
  instance_type = var.instance_type

  # セキュリティグループ
  vpc_security_group_ids = [aws_security_group.example.id]

  # IAM インスタンスプロファイル
  iam_instance_profile {
    name = aws_iam_instance_profile.example.name
  }

  # ユーザーデータ
  user_data = base64encode(templatefile("${path.module}/user_data.sh", {
    environment = var.environment
  }))

  # ブロックデバイスマッピング
  block_device_mappings {
    device_name = "/dev/sda1"
    ebs {
      volume_size           = 30
      volume_type           = "gp3"
      delete_on_termination = true
      encrypted             = true
    }
  }

  # タグ
  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "example-instance"
      Environment = var.environment
    }
  }

  # メタデータオプション（推奨）
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # IMDSv2 を強制
    http_put_response_hop_limit = 1
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

#### 1.2 ASG の起動設定を起動テンプレートに変更

```hcl
resource "aws_autoscaling_group" "example" {
  name                = "example-asg"
  vpc_zone_identifier = var.subnet_ids
  desired_capacity    = var.desired_capacity
  max_size            = var.max_size
  min_size            = var.min_size
  health_check_type   = "ELB"
  health_check_grace_period = 300

  # 旧: 起動設定
  # launch_configuration = aws_launch_configuration.example.name

  # 新: 起動テンプレート
  launch_template {
    id      = aws_launch_template.example.id
    version = "$Latest"  # または特定のバージョン番号
  }

  # ターゲットグループ
  target_group_arns = [aws_lb_target_group.example.arn]

  # タグ
  tag {
    key                 = "Name"
    value               = "example-instance"
    propagate_at_launch = true
  }

  tag {
    key                 = "Environment"
    value               = var.environment
    propagate_at_launch = true
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

#### 1.3 AMI ID の変数化
環境ごとに異なる AMI を指定できるよう変数化します。

```hcl
# variables.tf
variable "ami_id" {
  description = "AMI ID for EC2 instances"
  type        = string
}
```

```hcl
# terraform.tfvars (または環境ごとの tfvars ファイル)
# intg.tfvars
ami_id = "ami-xxxxxxxxxxxxxxxxx"  # 新しい AMI ID

# stg.tfvars
ami_id = "ami-yyyyyyyyyyyyyyyyy"

# prod.tfvars
ami_id = "ami-zzzzzzzzzzzzzzzzz"
```

#### 1.4 旧起動設定の削除（オプション）
起動テンプレート移行後、旧起動設定リソースはコメントアウトまたは削除できます（後方互換性のため、最初は残しておくことを推奨）。

---

### フェーズ 2: intg 環境での作業

#### 2.1 GitLab Pipeline で Terraform Plan 実行
1. GitLab リポジトリでブランチを作成（例: `feature/asg-migration-intg`）
2. Terraform コードを変更（上記フェーズ 1 の内容）
3. GitLab にプッシュ
4. Pipeline で `terraform plan` を実行し、変更内容を確認

**確認ポイント:**
- ASG の `launch_configuration` が削除され、`launch_template` が追加されているか
- 起動テンプレートの AMI ID が新しいものになっているか
- 既存インスタンスは置き換え対象として表示されない（ASG 設定のみ変更）

#### 2.2 GitLab Pipeline で Terraform Apply 実行
1. Plan の内容を確認後、`terraform apply` を実行
2. Apply が成功したことを確認

**結果:**
- ASG の設定が起動テンプレートに変更される
- 既存インスタンスはまだ旧 AMI のまま稼働

#### 2.3 AWS マネジメントコンソールでインスタンスリフレッシュ実行

##### 手順（AWS マネジメントコンソール）:

1. **AWS マネジメントコンソールにログイン**
   - intg 環境のアカウントにログイン

2. **EC2 > Auto Scaling > Auto Scaling グループ に移動**
   - 対象の ASG を選択

3. **「インスタンスのリフレッシュ」を開始**
   - ASG の詳細画面で「インスタンスのリフレッシュ」タブを選択
   - 「インスタンスリフレッシュを開始」ボタンをクリック

4. **設定を入力**
   - **最小正常率**: `90`（デフォルト）
     - 例: ASG の desired capacity が 4 の場合、最低 3.6 → 4 インスタンスが正常である必要がある
     - これにより、1 インスタンスずつ入れ替わる
   - **ウォームアッププロポーション**: `0` 秒（デフォルト）または適切な値
     - 新しいインスタンスがトラフィックを受け取る前の待機時間
   - **チェックポイント**: 必要に応じて設定（通常は不要）
   - **スキップマッチング**: 無効（デフォルト）

5. **リフレッシュを開始**
   - 「インスタンスリフレッシュを開始」をクリック

6. **進捗を監視**
   - 「インスタンスのリフレッシュ」タブでステータスを確認
   - インスタンスが 1 台ずつ終了され、新しい AMI で起動される
   - 所要時間: インスタンス数とヘルスチェック時間に依存（通常 10〜30 分）

##### 手順（AWS CLI）:

```bash
# インスタンスリフレッシュを開始
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name <ASG名> \
  --preferences MinHealthyPercentage=90,InstanceWarmup=0 \
  --region <リージョン>

# 進捗を確認
aws autoscaling describe-instance-refreshes \
  --auto-scaling-group-name <ASG名> \
  --region <リージョン>
```

#### 2.4 動作確認
1. **インスタンスの確認**
   - 新しいインスタンスが新しい AMI で起動していることを確認
   - EC2 > インスタンス で AMI ID を確認

2. **アプリケーションの動作確認**
   - エンドポイントにアクセスし、正常に動作することを確認
   - ログを確認し、エラーがないことを確認

3. **ALB/ELB のターゲットヘルスチェック確認**
   - すべてのインスタンスが "healthy" ステータスであることを確認

4. **監視メトリクスの確認**
   - CloudWatch でメトリクス（CPU、メモリ、リクエスト数など）を確認

#### 2.5 問題発生時のロールバック手順
もし問題が発生した場合:

1. **インスタンスリフレッシュのキャンセル**
   ```bash
   aws autoscaling cancel-instance-refresh \
     --auto-scaling-group-name <ASG名> \
     --region <リージョン>
   ```

2. **Terraform で旧設定に戻す**
   - Git で以前のコミットに戻す
   - `terraform apply` を実行

3. **手動で旧インスタンスをスケールアウト**（必要に応じて）
   - ASG の desired capacity を一時的に増やし、旧 AMI のインスタンスを起動

---

### フェーズ 3: stg 環境での作業

#### 3.1 intg での動作確認完了を確認
- intg 環境で問題なく動作していることを確認
- 関係者（チームリーダー、QA など）から承認を得る

#### 3.2 Terraform コードの stg 環境への適用
1. GitLab でマージリクエストを作成（intg ブランチ → stg ブランチ）
2. コードレビュー実施
3. マージ後、stg 環境用の Pipeline を実行

#### 3.3 stg 環境で Terraform Plan/Apply 実行
- フェーズ 2.1〜2.2 と同様の手順

#### 3.4 stg 環境でインスタンスリフレッシュ実行
- フェーズ 2.3 と同様の手順
- ただし、stg 環境は本番に近いため、より慎重に監視

#### 3.5 stg 環境で動作確認
- フェーズ 2.4 と同様の手順
- 必要に応じて、負荷テストや統合テストを実施

---

### フェーズ 4: prod 環境での作業

#### 4.1 stg での動作確認完了を確認
- stg 環境で問題なく動作していることを確認
- 関係者から最終承認を得る
- 変更管理チケット（Change Request）を発行（組織のポリシーに従う）

#### 4.2 Terraform コードの prod 環境への適用
1. GitLab でマージリクエストを作成（stg ブランチ → prod ブランチ）
2. コードレビュー実施
3. マージ後、prod 環境用の Pipeline を実行

#### 4.3 prod 環境で Terraform Plan/Apply 実行
- フェーズ 2.1〜2.2 と同様の手順
- **メンテナンスウィンドウ内で実施することを推奨**

#### 4.4 prod 環境でインスタンスリフレッシュ実行
- フェーズ 2.3 と同様の手順
- **最小正常率を高めに設定**（例: 95%）してダウンタイムを最小化
- **段階的にリフレッシュ**する場合:
  - 一部の ASG のみ先にリフレッシュ
  - 問題なければ残りの ASG をリフレッシュ

##### prod 環境での推奨設定:
```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name <ASG名> \
  --preferences MinHealthyPercentage=95,InstanceWarmup=300 \
  --region <リージョン>
```

#### 4.5 prod 環境で動作確認
- フェーズ 2.4 と同様の手順
- **リアルタイム監視を強化**:
  - CloudWatch ダッシュボードを常時監視
  - エラーログ、アクセスログをリアルタイムで確認
  - APM ツール（New Relic、Datadog など）でパフォーマンスを監視

#### 4.6 ロールバック計画
問題が発生した場合の緊急ロールバック手順を事前に準備:

1. インスタンスリフレッシュをキャンセル
2. Terraform で旧設定に戻す
3. 手動で ASG の desired capacity を調整し、旧 AMI インスタンスを起動
4. インシデント報告書を作成

---

## チェックリスト

### 事前準備
- [ ] 新しい AMI ID を各環境用に準備
- [ ] Terraform コードをレビュー
- [ ] バックアップ計画を確認
- [ ] ロールバック手順を文書化
- [ ] 関係者に作業スケジュールを通知

### intg 環境
- [ ] Terraform plan 実行・確認
- [ ] Terraform apply 実行
- [ ] インスタンスリフレッシュ実行
- [ ] 動作確認完了
- [ ] 承認取得

### stg 環境
- [ ] intg での動作確認完了
- [ ] Terraform plan 実行・確認
- [ ] Terraform apply 実行
- [ ] インスタンスリフレッシュ実行
- [ ] 動作確認完了
- [ ] 承認取得

### prod 環境
- [ ] stg での動作確認完了
- [ ] 変更管理チケット発行
- [ ] メンテナンスウィンドウ設定
- [ ] Terraform plan 実行・確認
- [ ] Terraform apply 実行
- [ ] インスタンスリフレッシュ実行（段階的に）
- [ ] 動作確認完了
- [ ] 完了報告

---

## トラブルシューティング

### インスタンスリフレッシュが失敗する
- **原因**: ヘルスチェックに失敗している
- **対処**:
  - ELB/ALB のターゲットグループでヘルスチェック設定を確認
  - インスタンスのログを確認（CloudWatch Logs、/var/log/）
  - セキュリティグループ、ネットワーク ACL を確認

### 新しいインスタンスがトラフィックを受け取らない
- **原因**: ターゲットグループに登録されていない
- **対処**:
  - ASG の設定で `target_group_arns` が正しく設定されているか確認
  - ターゲットグループの登録状況を確認

### Terraform apply でエラーが発生
- **原因**: リソースの依存関係が正しくない
- **対処**:
  - `terraform plan` で詳細なエラーメッセージを確認
  - 必要に応じて `depends_on` を追加

---

## 参考情報

### AWS ドキュメント
- [Auto Scaling グループの起動テンプレートの使用](https://docs.aws.amazon.com/ja_jp/autoscaling/ec2/userguide/launch-templates.html)
- [Auto Scaling グループのインスタンスリフレッシュ](https://docs.aws.amazon.com/ja_jp/autoscaling/ec2/userguide/instance-refresh.html)

### Terraform ドキュメント
- [aws_launch_template](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/launch_template)
- [aws_autoscaling_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/autoscaling_group)

---

## 完了報告テンプレート

```
# ASG 起動テンプレート移行 & OS イメージ更新 完了報告

## 実施日時
- intg: YYYY/MM/DD HH:MM
- stg: YYYY/MM/DD HH:MM
- prod: YYYY/MM/DD HH:MM

## 実施内容
- ASG を起動設定から起動テンプレートへ移行
- OS イメージを [旧 AMI ID] から [新 AMI ID] へ更新

## 結果
- すべての環境で正常に完了
- ダウンタイム: 0 分（インスタンスリフレッシュによるローリング更新）

## 動作確認結果
- アプリケーション動作: 正常
- ヘルスチェック: すべて "healthy"
- 監視メトリクス: 正常範囲内

## 今後の課題・改善点
- （あれば記載）
```

---

**作成日**: 2025-10-01
**バージョン**: 1.0
