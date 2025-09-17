# Amazon Linux 2 マイクロサービス OS移行手順書

## 1. 概要

### 1.1 移行背景
- Amazon Linux 2のサポート終了日：2026年6月30日
- セキュリティリスク軽減のため早期移行を推奨
- マイクロサービスアーキテクチャの最適化

### 1.2 移行先OS選定結果

| OS | サポート期限 | 推奨度 | 特徴 |
|---|---|---|---|
| **AlmaLinux 9** | 2032年まで（推奨） | ★★★ | RHEL互換、長期サポート、コンテナ対応良好 |
| Rocky Linux 9 | 2032年まで | ★★☆ | RHEL互換、コミュニティドリブン |
| Amazon Linux 2023 | 2029年6月30日 | ★☆☆ | AWS統合、リポジトリ制限あり |

**推奨：AlmaLinux 9**
- 10年間の長期サポート
- 豊富なサードパーティリポジトリ
- コンテナ技術との親和性が高い

## 2. 事前調査・準備フェーズ

### 2.1 現状システム調査

#### 2.1.1 インベントリ調査
```bash
# システム情報収集
cat /etc/os-release
uname -a
df -h
free -h
systemctl list-units --type=service --state=running
```

#### 2.1.2 インストール済みパッケージ調査
```bash
# パッケージリスト出力
rpm -qa > /tmp/installed_packages.txt
yum list installed > /tmp/yum_packages.txt
pip list > /tmp/python_packages.txt
npm list -g > /tmp/npm_packages.txt
```

#### 2.1.3 設定ファイル・データベックアップ
```bash
# 重要設定ファイルのバックアップ
tar -czf /tmp/config_backup_$(date +%Y%m%d).tar.gz \
  /etc/systemd/ \
  /etc/nginx/ \
  /etc/httpd/ \
  /etc/mysql/ \
  /etc/postgresql/ \
  /opt/app/config/
```

### 2.2 依存関係マッピング

#### 2.2.1 サービス依存関係調査
```bash
# サービス間通信の確認
netstat -tulpn
ss -tulpn
lsof -i

# AWS Application Discovery Service での調査（推奨）
aws application-discovery install-agent
aws application-discovery start-data-collection
```

#### 2.2.2 マイクロサービス構成確認
- [ ] サービスメッシュ構成（Istio、Envoy等）
- [ ] API Gateway設定
- [ ] ロードバランサー設定
- [ ] データベース接続情報
- [ ] 外部API依存関係
- [ ] ログ・監視システム連携

### 2.3 移行計画策定

#### 2.3.1 移行順序決定
1. **依存度の低いサービス**から開始
2. **読み取り専用サービス**
3. **書き込みサービス**
4. **核となるサービス**の順

#### 2.3.2 ダウンタイム計画
- [ ] メンテナンス窓時間の設定
- [ ] カナリアリリース計画
- [ ] ロールバック計画

## 3. テスト環境構築・検証フェーズ

### 3.1 テスト環境構築

#### 3.1.1 新OSインスタンス作成
```bash
# AlmaLinux 9 AMI選択
aws ec2 describe-images \
  --owners "679593333241" \
  --filters "Name=name,Values=AlmaLinux-9-*" \
  --query 'Images[*].[ImageId,Name,CreationDate]' \
  --output table

# インスタンス作成
aws ec2 run-instances \
  --image-id ami-xxxxxxxxx \
  --instance-type t3.medium \
  --key-name your-key \
  --security-group-ids sg-xxxxxxxxx \
  --subnet-id subnet-xxxxxxxxx
```

#### 3.1.2 基本環境セットアップ
```bash
# システム更新
sudo dnf update -y

# 必要なリポジトリ追加
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled crb

# Docker環境構築
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

### 3.2 アプリケーション移行テスト

#### 3.2.1 パッケージ互換性確認
```bash
# Amazon Linux 2 -> AlmaLinux 9 パッケージマッピング
# 主要パッケージの対応表作成
echo "amazon-linux-extras → dnf module"
echo "yum → dnf"
echo "systemd → systemd（同じ）"
```

#### 3.2.2 マイクロサービス個別テスト
```bash
# サービス起動テスト
sudo systemctl start your-microservice
sudo systemctl status your-microservice

# ヘルスチェック
curl -f http://localhost:8080/health || echo "Health check failed"

# ログ確認
sudo journalctl -u your-microservice -f
```

#### 3.2.3 統合テスト
- [ ] サービス間通信テスト
- [ ] データベース接続テスト  
- [ ] 外部API連携テスト
- [ ] パフォーマンステスト
- [ ] セキュリティテスト

## 4. 本番移行フェーズ

### 4.1 移行前チェックリスト

#### 4.1.1 バックアップ確認
- [ ] データベースバックアップ完了
- [ ] アプリケーションデータバックアップ完了
- [ ] 設定ファイルバックアップ完了
- [ ] AMIスナップショット作成完了

#### 4.1.2 通知・連絡
- [ ] 関係チームへの移行開始通知
- [ ] 監視システムでのアラート一時停止
- [ ] ユーザーへのメンテナンス通知

### 4.2 移行実行手順

#### 4.2.1 Blue-Green デプロイメント方式

**Step 1: Green環境構築**
```bash
# 新環境（Green）にAlmaLinux 9インスタンス作成
aws ec2 run-instances \
  --image-id ami-almalinux9 \
  --instance-type [現在と同サイズ] \
  --key-name [キー名] \
  --security-group-ids [セキュリティグループ] \
  --subnet-id [サブネット]

# アプリケーションデプロイ
# 設定ファイル配置
# サービス起動
```

**Step 2: トラフィック切り替え**
```bash
# ロードバランサーでトラフィック比率調整
# 10% → 50% → 100% の段階的切り替え

# Route 53 weighted routing利用例
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch file://traffic-switch.json
```

**Step 3: 監視・検証**
```bash
# メトリクス監視
aws logs filter-log-events \
  --log-group-name /aws/ec2/microservice \
  --start-time $(date -d '10 minutes ago' +%s)000

# アプリケーションメトリクス確認
curl -s http://new-instance:8080/metrics | grep error_rate
```

#### 4.2.2 ローリングアップデート方式

**Step 1: サービス単位の段階的移行**
```bash
# サービスA移行
# 1. 新インスタンス作成・設定
# 2. ロードバランサーに追加
# 3. 旧インスタンスをドレイン
# 4. 検証後、旧インスタンス削除

# サービスB移行（サービスA完了後）
# 同様の手順を繰り返し
```

### 4.3 移行後検証

#### 4.3.1 機能テスト
```bash
# APIエンドポイントテスト
curl -X GET http://api.example.com/health
curl -X POST http://api.example.com/test-data

# データベース接続テスト
mysql -h [DB_HOST] -u [USER] -p[PASS] -e "SELECT 1"
```

#### 4.3.2 パフォーマンステスト
```bash
# 負荷テスト（Apache Bench使用例）
ab -n 1000 -c 10 http://api.example.com/endpoint

# リソース使用率監視
top
htop
iotop
```

#### 4.3.3 ログ・監視確認
```bash
# エラーログ確認
sudo tail -f /var/log/messages
sudo journalctl -xe

# アプリケーションログ確認  
tail -f /opt/app/logs/application.log
```

## 5. 移行完了・運用移管フェーズ

### 5.1 旧環境クリーンアップ

#### 5.1.1 段階的リソース削除
```bash
# 1週間後: 旧インスタンスの停止
aws ec2 stop-instances --instance-ids i-xxxxxxxx

# 2週間後: 問題なければ削除
aws ec2 terminate-instances --instance-ids i-xxxxxxxx

# スナップショット保持期間: 3ヶ月
```

#### 5.1.2 DNSレコード更新
```bash
# 旧IP参照の削除
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch file://cleanup-dns.json
```

### 5.2 運用ドキュメント更新

#### 5.2.1 更新対象ドキュメント
- [ ] システム構成図
- [ ] 運用手順書
- [ ] 障害対応マニュアル
- [ ] バックアップ・リストア手順
- [ ] 監視設定ドキュメント

#### 5.2.2 新OS対応運用手順
```bash
# パッケージ更新手順（dnf利用）
sudo dnf check-update
sudo dnf update

# セキュリティ更新のみ
sudo dnf update --security

# サービス管理（systemctl）
sudo systemctl restart [service-name]
sudo systemctl reload [service-name]
```

## 6. リスク管理・ロールバック計画

### 6.1 想定リスクと対策

| リスク | 影響度 | 対策 |
|---|---|---|
| パッケージ非互換 | 高 | 事前テスト、代替パッケージ調査 |
| サービス間通信エラー | 高 | 段階的移行、ロールバック準備 |
| パフォーマンス劣化 | 中 | 負荷テスト、スケールアウト準備 |
| データ不整合 | 高 | データバックアップ、整合性チェック |

### 6.2 ロールバック手順

#### 6.2.1 緊急ロールバック（5分以内）
```bash
# ロードバランサーでトラフィック切り戻し
aws elbv2 modify-target-group \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=i-old-instance,Port=80

# DNS切り戻し（Route 53）
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch file://rollback-dns.json
```

#### 6.2.2 データロールバック
```bash
# データベースロールバック
mysql < backup_$(date +%Y%m%d).sql

# ファイルシステムロールバック  
aws ec2 create-snapshot --volume-id vol-backup
```

## 7. 移行チェックリスト

### 7.1 移行前チェックリスト

**計画段階**
- [ ] 移行対象システムの洗い出し完了
- [ ] 依存関係マッピング完了
- [ ] 移行順序決定
- [ ] ダウンタイム計画策定
- [ ] ロールバック計画策定
- [ ] 関係者への説明・承認取得

**準備段階**
- [ ] テスト環境構築完了
- [ ] 移行スクリプト作成・テスト完了
- [ ] バックアップ戦略確定
- [ ] 監視・アラート設定確認
- [ ] 緊急連絡体制構築

### 7.2 移行実行チェックリスト

**移行当日**
- [ ] バックアップ実行確認
- [ ] メンテナンス通知送信
- [ ] 監視アラート一時停止
- [ ] 移行作業実行
- [ ] 機能テスト実行
- [ ] パフォーマンステスト実行
- [ ] ロールバック判定
- [ ] 移行完了通知

### 7.3 移行後チェックリスト

**移行後1週間**
- [ ] 日次システムチェック
- [ ] エラーログ監視
- [ ] パフォーマンス監視
- [ ] ユーザーフィードバック収集

**移行後1ヶ月**
- [ ] 運用ドキュメント更新完了
- [ ] 旧環境クリーンアップ計画実行
- [ ] 移行作業振り返り実施
- [ ] 改善点整理・次回移行への反映

## 8. 補足資料

### 8.1 パッケージ対応表

| Amazon Linux 2 | AlmaLinux 9 | 備考 |
|---|---|---|
| amazon-linux-extras | dnf module | モジュール形式で提供 |
| yum | dnf | パッケージマネージャー変更 |
| python2 | python3 | Python2サポート終了 |
| mysql57 | mysql80 | バージョンアップが必要 |

### 8.2 移行支援ツール

- **AWS Application Migration Service (MGN)**: EC2移行自動化
- **AWS Database Migration Service (DMS)**: データベース移行
- **AWS System Manager**: パッチ管理・設定管理
- **Ansible**: 設定管理・デプロイ自動化

### 8.3 参考リンク

- [AlmaLinux 公式サイト](https://almalinux.org/)
- [AWS Migration Hub](https://aws.amazon.com/migration-hub/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)

---

**作成日**: $(date +%Y年%m月%d日)  
**版数**: v1.0  
**作成者**: システム移行チーム