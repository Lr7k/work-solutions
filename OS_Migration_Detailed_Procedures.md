# Amazon Linux 2 OS移行手順書（詳細版）
## AlmaLinux 9 & Amazon Linux 2023 対応

---

## 目次

1. [AlmaLinux 9 移行手順](#1-almalinux-9-移行手順)
2. [Amazon Linux 2023 移行手順](#2-amazon-linux-2023-移行手順)
3. [移行方式比較](#3-移行方式比較)
4. [トラブルシューティング](#4-トラブルシューティング)

---

## 1. AlmaLinux 9 移行手順

### 1.1 事前準備

#### 1.1.1 コマンドライン操作による事前調査

```bash
# 現在のシステム情報収集
echo "=== システム情報収集 ==="
cat /etc/os-release
uname -a
df -h
free -h

# インストール済みパッケージ調査
echo "=== パッケージ情報収集 ==="
rpm -qa | sort > /tmp/al2_packages.txt
yum list installed > /tmp/al2_yum_packages.txt

# サービス状態確認
echo "=== サービス状態確認 ==="
systemctl list-units --type=service --state=running > /tmp/al2_services.txt

# ネットワーク設定確認
echo "=== ネットワーク設定確認 ==="
ip addr show > /tmp/al2_network.txt
ss -tulpn > /tmp/al2_ports.txt

# 設定ファイルバックアップ
echo "=== 設定ファイルバックアップ ==="
sudo tar -czf /tmp/al2_config_backup_$(date +%Y%m%d_%H%M%S).tar.gz \
  /etc/systemd/ \
  /etc/nginx/ \
  /etc/httpd/ \
  /etc/mysql/ \
  /etc/postgresql/ \
  /etc/docker/ \
  /opt/ \
  /home/ 2>/dev/null

echo "事前調査完了。バックアップファイル: /tmp/al2_config_backup_*.tar.gz"
```

#### 1.1.2 AWS管理コンソール操作による事前準備

**手順1: EC2ダッシュボードでの現状確認**
1. AWS管理コンソールにログイン
2. EC2 > インスタンス を選択
3. 移行対象インスタンスを選択
4. 「詳細」タブで以下を記録：
   - インスタンスタイプ
   - セキュリティグループ
   - サブネット
   - IAMロール
   - タグ情報

**手順2: AMIスナップショット作成**
1. 移行対象インスタンスを右クリック
2. 「イメージとテンプレート」> 「イメージを作成」を選択
3. 設定項目：
   - イメージ名: `AL2-Backup-YYYYMMDD-HHMMSS`
   - イメージの説明: `Migration backup for [サービス名]`
   - 再起動なし: チェック
   - タグ追加: `Backup=true`, `MigrationDate=YYYY-MM-DD`
4. 「イメージを作成」をクリック

**手順3: EBSスナップショット作成**
1. EC2 > Elastic Block Store > スナップショット
2. 「スナップショットを作成」をクリック
3. 設定項目：
   - リソースタイプ: ボリューム
   - ボリューム: 対象ボリュームを選択
   - 説明: `AL2 Migration Backup - YYYY-MM-DD`
   - タグ: `Migration=Backup`
4. 「スナップショットを作成」をクリック

### 1.2 AlmaLinux 9 インスタンス作成

#### 1.2.1 コマンドライン操作（AWS CLI）

```bash
# AlmaLinux 9 AMI検索
echo "=== AlmaLinux 9 AMI検索 ==="
ALMA_AMI=$(aws ec2 describe-images \
  --owners "679593333241" \
  --filters "Name=name,Values=AlmaLinux-9-*" \
          "Name=state,Values=available" \
          "Name=architecture,Values=x86_64" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
  --output text)

echo "最新AlmaLinux 9 AMI: $ALMA_AMI"

# 現在のインスタンス情報取得
INSTANCE_ID="i-xxxxxxxxx"  # 移行対象インスタンスID
INSTANCE_INFO=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID)

# 設定情報抽出
INSTANCE_TYPE=$(echo $INSTANCE_INFO | jq -r '.Reservations[0].Instances[0].InstanceType')
SUBNET_ID=$(echo $INSTANCE_INFO | jq -r '.Reservations[0].Instances[0].SubnetId')
SECURITY_GROUPS=$(echo $INSTANCE_INFO | jq -r '.Reservations[0].Instances[0].SecurityGroups[].GroupId' | tr '\n' ' ')
KEY_NAME=$(echo $INSTANCE_INFO | jq -r '.Reservations[0].Instances[0].KeyName')
IAM_ROLE=$(echo $INSTANCE_INFO | jq -r '.Reservations[0].Instances[0].IamInstanceProfile.Arn // empty')

echo "=== インスタンス設定情報 ==="
echo "インスタンスタイプ: $INSTANCE_TYPE"
echo "サブネット: $SUBNET_ID"
echo "セキュリティグループ: $SECURITY_GROUPS"
echo "キー名: $KEY_NAME"
echo "IAMロール: $IAM_ROLE"

# AlmaLinux 9 インスタンス作成
echo "=== AlmaLinux 9 インスタンス作成 ==="
NEW_INSTANCE=$(aws ec2 run-instances \
  --image-id $ALMA_AMI \
  --instance-type $INSTANCE_TYPE \
  --key-name $KEY_NAME \
  --security-group-ids $SECURITY_GROUPS \
  --subnet-id $SUBNET_ID \
  --iam-instance-profile Name="EC2-Role" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=AlmaLinux9-Migration},{Key=Migration,Value=InProgress}]' \
  --user-data file://alma-userdata.sh \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "新しいインスタンス作成完了: $NEW_INSTANCE"

# インスタンス起動待機
echo "=== インスタンス起動待機 ==="
aws ec2 wait instance-running --instance-ids $NEW_INSTANCE
echo "インスタンス起動完了"

# パブリックIP取得
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $NEW_INSTANCE \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "パブリックIP: $PUBLIC_IP"
```

**userdata.sh ファイル作成**
```bash
cat > alma-userdata.sh << 'EOF'
#!/bin/bash
# AlmaLinux 9 初期設定スクリプト

# システム更新
dnf update -y

# 必要なリポジトリ追加
dnf install -y epel-release
dnf config-manager --set-enabled crb

# 基本パッケージインストール
dnf install -y \
  wget curl git vim \
  htop iotop \
  net-tools bind-utils \
  rsyslog crond \
  docker \
  python3-pip

# Docker設定
systemctl enable --now docker
usermod -aG docker ec2-user

# rsyslog, crond有効化
systemctl enable --now rsyslog
systemctl enable --now crond

# CloudWatch Agent インストール
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
rpm -U amazon-cloudwatch-agent.rpm

# 完了ログ
echo "AlmaLinux 9 初期設定完了: $(date)" >> /var/log/migration.log
EOF
```

#### 1.2.2 AWS管理コンソール操作

**手順1: EC2インスタンス作成**
1. EC2 > インスタンス > 「インスタンスを起動」
2. **名前とタグ**:
   - 名前: `AlmaLinux9-Migration-[サービス名]`
   - タグ追加: `Migration=InProgress`, `OS=AlmaLinux9`

3. **アプリケーションおよびOSイメージ**:
   - 「その他のAMIを参照」をクリック
   - 「コミュニティAMI」タブを選択
   - 検索: `AlmaLinux-9`
   - 最新のAlmaLinux 9 AMIを選択

4. **インスタンスタイプ**:
   - 現在と同じインスタンスタイプを選択

5. **キーペア**:
   - 既存のキーペアを選択

6. **ネットワーク設定**:
   - VPC: 現在と同じVPCを選択
   - サブネット: 現在と同じサブネットを選択
   - セキュリティグループ: 既存のセキュリティグループを選択

7. **ストレージを設定**:
   - 現在と同じストレージサイズを設定

8. **高度な詳細**:
   - IAMインスタンスプロファイル: 現在と同じロールを選択
   - ユーザーデータ: 上記のuserdata.shの内容を貼り付け

9. 「インスタンスを起動」をクリック

### 1.3 アプリケーション移行

#### 1.3.1 コマンドライン操作による移行

```bash
# 新しいAlmaLinux 9インスタンスに接続
NEW_INSTANCE_IP="xxx.xxx.xxx.xxx"  # 新インスタンスのIP
ssh -i ~/.ssh/your-key.pem ec2-user@$NEW_INSTANCE_IP

# === AlmaLinux 9 での環境構築 ===
echo "=== システム環境構築開始 ==="

# dnfコマンドでパッケージ管理
sudo dnf update -y

# Amazon Linux 2のyum extras相当をdnf moduleで対応
sudo dnf module list
sudo dnf module install -y nodejs:18
sudo dnf module install -y python39

# 必要なパッケージインストール
sudo dnf install -y \
  nginx \
  mysql-server \
  redis \
  java-11-openjdk \
  php-fpm

# サービス有効化
sudo systemctl enable --now nginx
sudo systemctl enable --now mysqld
sudo systemctl enable --now redis

# === アプリケーションデータ移行 ===
echo "=== アプリケーションデータ移行 ==="

# 旧サーバーからの設定ファイル同期
OLD_INSTANCE_IP="yyy.yyy.yyy.yyy"  # 旧インスタンスのIP

# 設定ファイル取得
scp -i ~/.ssh/your-key.pem \
  ec2-user@$OLD_INSTANCE_IP:/tmp/al2_config_backup_*.tar.gz \
  /tmp/

# バックアップ展開
cd /tmp
tar -xzf al2_config_backup_*.tar.gz

# 設定ファイル適用（AlmaLinux 9対応）
sudo cp -r etc/nginx/* /etc/nginx/ 2>/dev/null || true
sudo cp -r etc/systemd/system/* /etc/systemd/system/ 2>/dev/null || true

# systemdデーモンリロード
sudo systemctl daemon-reload

# アプリケーションファイル同期
rsync -avz -e "ssh -i ~/.ssh/your-key.pem" \
  ec2-user@$OLD_INSTANCE_IP:/opt/app/ \
  /opt/app/

# 権限設定
sudo chown -R ec2-user:ec2-user /opt/app/
sudo chmod +x /opt/app/bin/*

# === データベース移行 ===
echo "=== データベース移行 ==="

# MySQL初期設定
sudo mysql_secure_installation

# データベースダンプ取得（旧サーバーで実行）
ssh -i ~/.ssh/your-key.pem ec2-user@$OLD_INSTANCE_IP \
  "mysqldump -u root -p --all-databases > /tmp/mysql_backup.sql"

# ダンプファイル取得
scp -i ~/.ssh/your-key.pem \
  ec2-user@$OLD_INSTANCE_IP:/tmp/mysql_backup.sql \
  /tmp/

# データベース復元
mysql -u root -p < /tmp/mysql_backup.sql

# === サービス起動・確認 ===
echo "=== サービス起動・確認 ==="

# Webサーバー起動
sudo systemctl start nginx
sudo systemctl status nginx

# アプリケーションサービス起動
sudo systemctl start your-app-service
sudo systemctl status your-app-service

# ヘルスチェック
curl -f http://localhost/health || echo "ヘルスチェック失敗"
curl -f http://localhost:8080/api/status || echo "APIチェック失敗"

echo "=== 移行完了 ==="
```

#### 1.3.2 AWS管理コンソール操作による移行

**手順1: Systems Manager セッションマネージャー接続**
1. EC2 > インスタンス > 新しいAlmaLinux 9インスタンスを選択
2. 「接続」ボタンをクリック
3. 「Session Manager」タブを選択
4. 「接続」をクリック

**手順2: Application Load Balancer設定更新**
1. EC2 > ロードバランサー
2. 対象のALBを選択
3. 「リスナー」タブ > リスナールールを選択
4. 「ターゲットグループ」をクリック
5. 新しいターゲットグループを作成：
   - 名前: `almalinux9-migration-tg`
   - プロトコル: HTTP/HTTPS
   - ポート: 80/443
   - VPC: 同じVPCを選択
6. 「次へ」> ターゲット登録で新しいAlmaLinux 9インスタンスを追加
7. 「ターゲットグループを作成」

**手順3: Route 53 weighted routing設定**
1. Route 53 > ホストゾーン > 対象ドメインを選択
2. 「レコードを作成」をクリック
3. 設定項目：
   - レコード名: api (例)
   - レコードタイプ: A
   - エイリアス: はい
   - トラフィックのルーティング先: Application Load Balancer
   - リージョン: 適切なリージョンを選択
   - ロードバランサー: 新しいALBを選択
   - ルーティングポリシー: 重み付け
   - 重み: 10 (10%のトラフィック)
   - セットID: almalinux9-test
4. 「レコードを作成」をクリック

**手順4: CloudWatch監視設定**
1. CloudWatch > メトリクス > すべてのメトリクス
2. 「EC2」> 「インスタンス別」
3. 新しいインスタンスのメトリクス選択：
   - CPUUtilization
   - NetworkIn/Out
   - DiskReadOps/WriteOps
4. アラーム作成：
   - メトリクス: CPUUtilization
   - 条件: > 80%
   - 期間: 5分間
   - アクション: SNS通知設定

### 1.4 トラフィック切り替え

#### 1.4.1 コマンドライン操作

```bash
# === 段階的トラフィック切り替え ===
echo "=== 段階的トラフィック切り替え開始 ==="

# Route 53レコード更新用JSON作成
cat > route53-update.json << EOF
{
    "Comment": "AlmaLinux 9 Migration - Traffic Shift",
    "Changes": [
        {
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "api.yourdomain.com",
                "Type": "A",
                "SetIdentifier": "almalinux9",
                "Weight": 50,
                "AliasTarget": {
                    "DNSName": "your-new-alb.us-west-2.elb.amazonaws.com",
                    "EvaluateTargetHealth": true,
                    "HostedZoneId": "Z1D633PJN98FT9"
                }
            }
        },
        {
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "api.yourdomain.com",
                "Type": "A",
                "SetIdentifier": "amazonlinux2",
                "Weight": 50,
                "AliasTarget": {
                    "DNSName": "your-old-alb.us-west-2.elb.amazonaws.com",
                    "EvaluateTargetHealth": true,
                    "HostedZoneId": "Z1D633PJN98FT9"
                }
            }
        }
    ]
}
EOF

# 50%トラフィック切り替え
aws route53 change-resource-record-sets \
    --hosted-zone-id Z123456789 \
    --change-batch file://route53-update.json

# 変更確認
aws route53 get-change --id /change/C123456789

# === 監視・検証 ===
echo "=== 監視・検証 ==="

# CloudWatchメトリクス監視
watch -n 30 'aws cloudwatch get-metric-statistics \
    --namespace AWS/EC2 \
    --metric-name CPUUtilization \
    --dimensions Name=InstanceId,Value='$NEW_INSTANCE' \
    --start-time $(date -u -d "5 minutes ago" +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average \
    --query "Datapoints[0].Average"'

# アプリケーションメトリクス確認
curl -s http://api.yourdomain.com/metrics | grep -E "(error_rate|response_time|throughput)"

# ログ監視
ssh -i ~/.ssh/your-key.pem ec2-user@$NEW_INSTANCE_IP \
    "sudo tail -f /var/log/nginx/access.log | grep -E '(4[0-9][0-9]|5[0-9][0-9])'"
```

#### 1.4.2 AWS管理コンソール操作

**手順1: ALB Target Group重み付け調整**
1. EC2 > ターゲットグループ
2. 既存のターゲットグループを選択
3. 「ターゲット」タブで新しいインスタンスを追加
4. 「ヘルスチェック」で正常性確認
5. 正常確認後、「編集」で重み付け調整：
   - 旧インスタンス: 重み 50
   - 新インスタンス: 重み 50

**手順2: Route 53 重み付けルーティング更新**
1. Route 53 > ホストゾーン > レコード選択
2. 既存レコードを編集：
   - ルーティングポリシー: 重み付け
   - 重み: 50 → 20 → 0 (段階的に削減)
3. 新しいレコードの重み: 50 → 80 → 100 (段階的に増加)

**手順3: CloudWatch ダッシュボード作成**
1. CloudWatch > ダッシュボード > 「ダッシュボードの作成」
2. ダッシュボード名: `AlmaLinux9-Migration-Monitor`
3. ウィジェット追加：
   - EC2 CPU使用率 (旧・新インスタンス比較)
   - ALB レスポンス時間
   - ALB エラー率
   - Route 53 クエリ数
4. 「ダッシュボードを作成」

---

## 2. Amazon Linux 2023 移行手順

### 2.1 事前準備

#### 2.1.1 コマンドライン操作による事前調査

```bash
# === Amazon Linux 2023向け事前調査 ===
echo "=== Amazon Linux 2023移行向け事前調査 ==="

# パッケージ互換性チェック
echo "=== パッケージ互換性チェック ==="
yum list installed | grep -E "(mysql|php|nginx|httpd|python)" > /tmp/al2_packages_check.txt

# Amazon Linux Extras確認
amazon-linux-extras list > /tmp/al2_extras.txt

# カスタムリポジトリ確認
yum repolist > /tmp/al2_repos.txt

# Python2依存関係確認
echo "=== Python2依存関係確認 ==="
find /opt /usr/local -name "*.py" -exec grep -l "python2\|#!/usr/bin/python[^3]" {} \; > /tmp/python2_dependencies.txt

# systemd timer確認
echo "=== systemd timer確認 ==="
systemctl list-timers > /tmp/systemd_timers.txt

# cron job確認
echo "=== cron job確認 ==="
crontab -l > /tmp/user_crontab.txt 2>/dev/null || echo "No user crontab"
sudo crontab -l > /tmp/root_crontab.txt 2>/dev/null || echo "No root crontab"
ls -la /etc/cron.d/ > /tmp/cron_d.txt

echo "事前調査完了。AL2023移行における注意点を確認してください。"
```

#### 2.1.2 AWS管理コンソール操作による準備

**手順1: Migration Hub での依存関係分析**
1. AWS Migration Hub にアクセス
2. 「発見」> 「データコレクター」
3. 「Application Discovery Agent」をダウンロード・インストール
4. 「データ収集の開始」で7日間のデータ収集開始
5. 「アプリケーション」タブで依存関係マップ確認

**手順2: AWS Config による設定ベースライン記録**
1. AWS Config > 「開始方法」
2. 設定レコーダー作成：
   - 名前: `AL2-Migration-Baseline`
   - ロール: 新しいロールを作成
   - S3バケット: 新しいバケットを作成
3. 「ルール」で以下を有効化：
   - ec2-instance-managed-by-ssm
   - ec2-security-group-attached-to-eni
   - ec2-stopped-instance

### 2.2 Amazon Linux 2023 インスタンス作成

#### 2.2.1 コマンドライン操作（AWS CLI）

```bash
# === Amazon Linux 2023 AMI検索・インスタンス作成 ===
echo "=== Amazon Linux 2023 AMI検索 ==="

# 最新AL2023 AMI取得
AL2023_AMI=$(aws ec2 describe-images \
  --owners "amazon" \
  --filters "Name=name,Values=al2023-ami-2023*" \
          "Name=state,Values=available" \
          "Name=architecture,Values=x86_64" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
  --output text)

echo "最新Amazon Linux 2023 AMI: $AL2023_AMI"

# userdata.shファイル作成（AL2023対応）
cat > al2023-userdata.sh << 'EOF'
#!/bin/bash
# Amazon Linux 2023 初期設定スクリプト

# システム更新
dnf update -y

# rsyslog, crond手動インストール（AL2023では標準で含まれない）
dnf install -y rsyslog cronie

# 基本パッケージインストール
dnf install -y \
  wget curl git vim \
  htop iotop \
  net-tools bind-utils \
  docker \
  python3-pip

# MySQL（AL2023では公式リポジトリにMySQLなし）
# MySQL公式リポジトリ追加
dnf install -y https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm
dnf install -y mysql-community-server

# Docker設定
systemctl enable --now docker
usermod -aG docker ec2-user

# rsyslog, crond有効化
systemctl enable --now rsyslog
systemctl enable --now crond

# Node.js（AL2023では標準リポジトリから）
dnf install -y nodejs npm

# CloudWatch Agent
dnf install -y amazon-cloudwatch-agent

# 完了ログ
echo "Amazon Linux 2023 初期設定完了: $(date)" >> /var/log/migration.log
EOF

# Amazon Linux 2023インスタンス作成
echo "=== Amazon Linux 2023 インスタンス作成 ==="
AL2023_INSTANCE=$(aws ec2 run-instances \
  --image-id $AL2023_AMI \
  --instance-type $INSTANCE_TYPE \
  --key-name $KEY_NAME \
  --security-group-ids $SECURITY_GROUPS \
  --subnet-id $SUBNET_ID \
  --iam-instance-profile Name="EC2-Role" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=AL2023-Migration},{Key=Migration,Value=InProgress}]' \
  --user-data file://al2023-userdata.sh \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Amazon Linux 2023インスタンス作成: $AL2023_INSTANCE"

# インスタンス起動待機
aws ec2 wait instance-running --instance-ids $AL2023_INSTANCE

# パブリックIP取得
AL2023_PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $AL2023_INSTANCE \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "Amazon Linux 2023 パブリックIP: $AL2023_PUBLIC_IP"
```

#### 2.2.2 AWS管理コンソール操作

**手順1: EC2インスタンス作成（AL2023）**
1. EC2 > インスタンス > 「インスタンスを起動」
2. **名前とタグ**:
   - 名前: `AL2023-Migration-[サービス名]`
   - タグ: `Migration=InProgress`, `OS=AmazonLinux2023`

3. **アプリケーションおよびOSイメージ**:
   - 「Amazon Linux」を選択
   - 「Amazon Linux 2023 AMI」を選択（最新版）

4. **インスタンスタイプ・ネットワーク設定**: 
   - AlmaLinux 9と同様の設定を適用

5. **高度な詳細**のユーザーデータ:
```bash
#!/bin/bash
# Amazon Linux 2023 初期設定
dnf update -y
dnf install -y rsyslog cronie docker
systemctl enable --now rsyslog crond docker
usermod -aG docker ec2-user

# MySQL公式リポジトリ（AL2023対応）
dnf install -y https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm
dnf install -y mysql-community-server

echo "AL2023 setup completed" >> /var/log/migration.log
```

### 2.3 アプリケーション移行（AL2023固有対応）

#### 2.3.1 コマンドライン操作

```bash
# Amazon Linux 2023への接続
ssh -i ~/.ssh/your-key.pem ec2-user@$AL2023_PUBLIC_IP

# === AL2023固有の設定変更 ===
echo "=== Amazon Linux 2023 固有設定 ==="

# dnfコマンドの使用（yum→dnfへの変更）
sudo dnf update -y

# Amazon Linux Extras → dnf repository/moduleへの移行
echo "=== リポジトリ設定 ==="

# Python3（標準でインストール済み）
python3 --version

# Node.js（標準リポジトリから）
sudo dnf install -y nodejs npm

# PHP（標準リポジトリから）
sudo dnf install -y php php-fpm php-mysql php-json

# Nginx（標準リポジトリから）
sudo dnf install -y nginx

# === AL2からAL2023への設定移行 ===
echo "=== 設定ファイル移行（AL2023対応） ==="

# 旧設定ファイル取得
scp -i ~/.ssh/your-key.pem \
  ec2-user@$OLD_INSTANCE_IP:/tmp/al2_config_backup_*.tar.gz \
  /tmp/

# バックアップ展開
cd /tmp
tar -xzf al2_config_backup_*.tar.gz

# systemd設定の移行（AL2023互換性チェック）
echo "=== systemd設定移行 ==="
for service_file in etc/systemd/system/*.service; do
    if [ -f "$service_file" ]; then
        echo "Checking $service_file for AL2023 compatibility..."
        # AL2023でPython2参照を排除
        sed -i 's|/usr/bin/python|/usr/bin/python3|g' "$service_file"
        sed -i 's|python2|python3|g' "$service_file"
        sudo cp "$service_file" "/etc/systemd/system/"
    fi
done

# systemdデーモンリロード
sudo systemctl daemon-reload

# cron→systemd timer移行（推奨）
echo "=== cron→systemd timer移行 ==="
if [ -f "/tmp/user_crontab.txt" ]; then
    echo "既存cronジョブをsystemd timerに移行することを推奨します"
    cat /tmp/user_crontab.txt
fi

# === アプリケーション固有の移行 ===
echo "=== アプリケーション移行 ==="

# アプリケーションディレクトリ同期
rsync -avz -e "ssh -i ~/.ssh/your-key.pem" \
  ec2-user@$OLD_INSTANCE_IP:/opt/app/ \
  /opt/app/

# Python2→Python3移行対応
echo "=== Python2→Python3移行対応 ==="
find /opt/app -name "*.py" -exec sed -i '1s|#!/usr/bin/python$|#!/usr/bin/python3|' {} \;
find /opt/app -name "*.py" -exec sed -i '1s|#!/usr/bin/env python$|#!/usr/bin/env python3|' {} \;

# requirements.txt対応（Python3互換パッケージ）
if [ -f "/opt/app/requirements.txt" ]; then
    # Python3互換性チェック
    pip3 install --dry-run -r /opt/app/requirements.txt
fi

# Node.js パッケージ更新
if [ -f "/opt/app/package.json" ]; then
    cd /opt/app
    npm install
    npm audit fix
fi

# === サービス起動・設定 ===
echo "=== サービス起動・設定 ==="

# MySQL設定（AL2023対応）
sudo systemctl start mysqld
sudo systemctl enable mysqld

# MySQL初期パスワード取得（AL2023では異なる可能性）
TEMP_PASSWORD=$(sudo grep 'temporary password' /var/log/mysqld.log | awk '{print $NF}')
echo "MySQL temporary password: $TEMP_PASSWORD"

# アプリケーションサービス起動
sudo systemctl start your-app-service
sudo systemctl enable your-app-service

# Nginx起動
sudo systemctl start nginx
sudo systemctl enable nginx

# ヘルスチェック
curl -f http://localhost/health
curl -f http://localhost:8080/api/status

echo "=== AL2023移行完了 ==="
```

#### 2.3.2 AWS管理コンソール操作

**手順1: Systems Manager Parameter Store設定更新**
1. Systems Manager > Parameter Store
2. 新しいパラメータ作成：
   - 名前: `/app/config/python-version`
   - 値: `python3`
   - 説明: `AL2023 migration - Python version update`

**手順2: CloudFormation テンプレート更新**
1. CloudFormation > スタック > 対象スタックを選択
2. 「更新」をクリック
3. テンプレート内のImageIdを最新AL2023 AMIに更新
4. UserDataスクリプトをAL2023対応版に更新
5. 「変更セットの作成」> 「実行」

**手順3: CodeDeploy アプリケーション更新**
1. CodeDeploy > アプリケーション > 対象アプリを選択
2. 「デプロイグループ」を編集
3. AL2023インスタンスをターゲットに追加
4. デプロイ設定で段階的デプロイを設定

### 2.4 AL2023特有の注意点対応

#### 2.4.1 コマンドライン操作

```bash
# === AL2023特有の制限事項対応 ===
echo "=== AL2023制限事項対応 ==="

# EPELリポジトリ使用不可対応
echo "EPEL代替パッケージ確認:"
# 必要なパッケージをAL2023標準リポジトリから選択

# Remiリポジトリ使用不可対応
echo "Remi代替対応:"
# PHPの最新版はAL2023標準リポジトリから利用

# MySQL公式リポジトリ設定（AL2023で必須）
echo "=== MySQL公式リポジトリ設定 ==="
sudo dnf install -y https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql-community-server mysql-community-client

# rsyslog・crondの手動インストール（AL2023では含まれない）
echo "=== 必須サービス手動インストール ==="
sudo dnf install -y rsyslog cronie
sudo systemctl enable --now rsyslog
sudo systemctl enable --now crond

# Amazon Linux Extras代替機能確認
echo "=== AL Extras代替確認 ==="
dnf module list available | grep -E "(nodejs|python|php)"

# === セキュリティ設定強化（AL2023推奨） ===
echo "=== セキュリティ設定強化 ==="

# SELinux設定確認（AL2023ではEnforcingがデフォルト）
getenforce
sudo setsebool -P httpd_can_network_connect 1

# FirewallD設定（AL2023ではfirewalldがデフォルト）
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

echo "=== AL2023特有対応完了 ==="
```

---

## 3. 移行方式比較

### 3.1 総合比較表

| 項目 | AlmaLinux 9 | Amazon Linux 2023 |
|------|-------------|-------------------|
| **サポート期間** | 2032年まで（10年） | 2029年6月30日（6年） |
| **パッケージ管理** | dnf + EPEL/Remi対応 | dnf（EPELなし） |
| **MySQL** | 標準リポジトリあり | 公式リポジトリ必須 |
| **rsyslog/crond** | 標準インストール | 手動インストール必須 |
| **コンテナ技術** | 優秀 | 良好 |
| **AWS統合** | 標準的 | 最適化済み |
| **移行難易度** | 中程度 | 高（制限事項多い） |
| **企業向け安定性** | 高（RHEL互換） | 中（AWS特化） |

### 3.2 コマンド対応表

| 作業内容 | Amazon Linux 2 | AlmaLinux 9 | Amazon Linux 2023 |
|----------|-----------------|-------------|-------------------|
| パッケージ更新 | `yum update` | `dnf update` | `dnf update` |
| MySQL インストール | `yum install mysql` | `dnf install mysql-server` | MySQL公式リポジトリ必要 |
| PHP インストール | `amazon-linux-extras install php7.4` | `dnf install php` | `dnf install php` |
| Node.js インストール | `amazon-linux-extras install nodejs` | `dnf module install nodejs:18` | `dnf install nodejs` |
| EPEL リポジトリ | `amazon-linux-extras install epel` | `dnf install epel-release` | 利用不可 |

### 3.3 移行推奨判定フローチャート

```
マイクロサービス移行先OS選定
│
├─ 長期サポート重視？
│  ├─ Yes → AlmaLinux 9 推奨
│  └─ No → 次の質問へ
│
├─ AWS サービス統合重視？
│  ├─ Yes → Amazon Linux 2023
│  └─ No → AlmaLinux 9
│
├─ サードパーティリポジトリ必要？
│  ├─ Yes → AlmaLinux 9 推奨
│  └─ No → Amazon Linux 2023
│
└─ コンテナ・Kubernetes中心？
   ├─ Yes → AlmaLinux 9 推奨
   └─ No → どちらでも可
```

---

## 4. トラブルシューティング

### 4.1 共通問題

#### 4.1.1 パッケージ依存関係エラー

**問題**: パッケージインストール時の依存関係エラー

**AlmaLinux 9 対応**:
```bash
# 依存関係確認
dnf deplist package-name

# 強制インストール（非推奨）
dnf install --nobest package-name

# 代替パッケージ検索
dnf search alternative-package
```

**Amazon Linux 2023 対応**:
```bash
# モジュール確認
dnf module list | grep package-name

# モジュールインストール
dnf module install package-name:version
```

#### 4.1.2 サービス起動エラー

**問題**: systemdサービスが起動しない

**共通対応**:
```bash
# 詳細エラー確認
systemctl status service-name
journalctl -xe -u service-name

# 設定ファイル構文チェック
systemd-analyze verify /etc/systemd/system/service-name.service

# SELinux関連確認
sealert -a /var/log/audit/audit.log
```

### 4.2 OS固有問題

#### 4.2.1 AlmaLinux 9 固有

**問題**: RHEL互換性問題

**対応**:
```bash
# CentOS Stream リポジトリ追加（必要に応じて）
dnf install centos-release-stream

# パッケージの代替確認
dnf provides */file-path
```

#### 4.2.2 Amazon Linux 2023 固有

**問題**: EPELパッケージが利用できない

**対応**:
```bash
# 代替パッケージ検索
dnf search package-name

# ソースからビルド
dnf install gcc make
wget source-package.tar.gz
./configure && make && make install
```

**問題**: MySQL公式リポジトリ設定エラー

**対応**:
```bash
# リポジトリ確認
dnf repolist | grep mysql

# GPGキー問題対応
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022

# 強制インストール
dnf install mysql-community-server --nogpgcheck
```

---

**文書情報**
- 作成日: 2024年12月XX日
- 版数: v2.0 詳細版
- 対象: Amazon Linux 2 → AlmaLinux 9 / Amazon Linux 2023 移行