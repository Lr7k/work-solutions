# AWS Auto Scaling Group: 起動設定から起動テンプレートへの移行手順

## 概要
AWS Auto Scaling Groupで使用されている起動設定（Launch Configuration）は非推奨となっており、起動テンプレート（Launch Template）への移行が推奨されています。

この手順書では、AWS CLIとAWSマネジメントコンソールの両方での移行方法を説明します。

## 前提条件
- AWS CLIまたはAWSマネジメントコンソールへのアクセス権限
- 適切なIAMロールと権限（ec2:*、autoscaling:*）
- 既存のAuto Scaling Groupの設定情報

# 方法1: AWS CLIを使用した移行

## 1. 既存起動設定の情報収集

### 1.1 起動設定の詳細確認
```bash
# 起動設定一覧の取得
aws autoscaling describe-launch-configurations

# 特定の起動設定の詳細取得
aws autoscaling describe-launch-configurations --launch-configuration-names [LAUNCH_CONFIG_NAME]
```

### 1.2 Auto Scaling Groupの確認
```bash
# Auto Scaling Group一覧の取得
aws autoscaling describe-auto-scaling-groups

# 特定のAuto Scaling Groupの詳細取得
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names [ASG_NAME]
```

### 1.3 収集すべき情報
- AMI ID
- インスタンスタイプ
- セキュリティグループ
- キーペア
- IAMインスタンスプロファイル
- ユーザーデータ
- ブロックデバイスマッピング
- インスタンス監視設定
- 配置情報

## 2. 起動テンプレートの作成

### 2.1 JSONファイルの準備
起動設定の情報を基に起動テンプレート用のJSONファイルを作成：

```json
{
    "LaunchTemplateName": "my-launch-template",
    "LaunchTemplateData": {
        "ImageId": "ami-xxxxxxxxx",
        "InstanceType": "t3.micro",
        "KeyName": "my-key-pair",
        "SecurityGroupIds": ["sg-xxxxxxxxx"],
        "IamInstanceProfile": {
            "Name": "my-instance-profile"
        },
        "UserData": "base64-encoded-user-data",
        "BlockDeviceMappings": [
            {
                "DeviceName": "/dev/xvda",
                "Ebs": {
                    "VolumeSize": 20,
                    "VolumeType": "gp3",
                    "DeleteOnTermination": true
                }
            }
        ],
        "Monitoring": {
            "Enabled": true
        },
        "TagSpecifications": [
            {
                "ResourceType": "instance",
                "Tags": [
                    {
                        "Key": "Name",
                        "Value": "my-instance"
                    }
                ]
            }
        ]
    }
}
```

### 2.2 起動テンプレートの作成
```bash
# JSON ファイルを使用して起動テンプレートを作成
aws ec2 create-launch-template --cli-input-json file://launch-template.json

# または直接コマンドラインで作成
aws ec2 create-launch-template \
    --launch-template-name my-launch-template \
    --launch-template-data '{"ImageId":"ami-xxxxxxxxx","InstanceType":"t3.micro",...}'
```

## 3. Auto Scaling Groupの更新

### 3.1 起動テンプレートへの切り替え
```bash
# Auto Scaling Groupを起動テンプレートに更新
aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name [ASG_NAME] \
    --launch-template LaunchTemplateName=[TEMPLATE_NAME],Version='$Latest'
```

### 3.2 起動設定の削除（オプション）
```bash
# 起動設定を削除（Auto Scaling Groupから参照されていない場合のみ）
aws autoscaling delete-launch-configuration \
    --launch-configuration-name [LAUNCH_CONFIG_NAME]
```

## 4. 検証・テスト

### 4.1 設定の確認
```bash
# 更新されたAuto Scaling Groupの確認
aws autoscaling describe-auto-scaling-groups \
    --auto-scaling-group-names [ASG_NAME]
```

### 4.2 インスタンス起動テスト
```bash
# スケールアウトテスト
aws autoscaling set-desired-capacity \
    --auto-scaling-group-name [ASG_NAME] \
    --desired-capacity [CURRENT_CAPACITY + 1] \
    --honor-cooldown
```

### 4.3 新しいインスタンスの確認
- インスタンスが正常に起動するか
- 必要なタグが適用されているか
- セキュリティグループが正しく設定されているか
- ユーザーデータが正常に実行されているか

## 5. ロールバック手順

### 5.1 緊急時のロールバック
```bash
# 元の起動設定に戻す
aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name [ASG_NAME] \
    --launch-configuration-name [ORIGINAL_LAUNCH_CONFIG_NAME]
```

### 5.2 起動テンプレートの削除
```bash
# 問題のある起動テンプレートを削除
aws ec2 delete-launch-template \
    --launch-template-name [TEMPLATE_NAME]
```

## 6. 移行後の最適化

### 6.1 起動テンプレートのバージョン管理
- 変更履歴の追跡
- バージョン番号の管理
- デフォルトバージョンの設定

### 6.2 Mixed Instance Policyの活用
```json
{
    "MixedInstancesPolicy": {
        "LaunchTemplate": {
            "LaunchTemplateSpecification": {
                "LaunchTemplateName": "my-launch-template",
                "Version": "$Latest"
            },
            "Overrides": [
                {
                    "InstanceType": "t3.micro"
                },
                {
                    "InstanceType": "t3.small"
                }
            ]
        },
        "InstancesDistribution": {
            "OnDemandPercentage": 50,
            "SpotAllocationStrategy": "diversified"
        }
    }
}
```

# 方法2: AWSマネジメントコンソールを使用した移行

## 1. 既存起動設定の情報収集（画面操作）

### 1.1 起動設定の詳細確認
1. **AWSマネジメントコンソール**にログイン
2. **EC2サービス**を選択
3. 左側メニューから**Auto Scaling** → **起動設定**を選択
4. 対象の起動設定をクリックして詳細画面を開く
5. 以下の情報をメモまたはスクリーンショット保存：
   - AMI ID
   - インスタンスタイプ
   - セキュリティグループ
   - キーペア名
   - IAMロール
   - ユーザーデータ
   - ストレージ設定
   - 詳細監視設定

### 1.2 Auto Scaling Groupの確認
1. 左側メニューから**Auto Scaling** → **Auto Scaling グループ**を選択
2. 対象のASGをクリックして詳細画面を開く
3. **詳細**タブで現在の起動設定を確認
4. **インスタンス**タブで現在稼働中のインスタンスを確認

## 2. 起動テンプレートの作成（画面操作）

### 2.1 起動テンプレートの新規作成
1. **EC2サービス**の左側メニューから**インスタンス** → **起動テンプレート**を選択
2. **起動テンプレートを作成**ボタンをクリック
3. 基本設定を入力：
   - **起動テンプレート名**: わかりやすい名前を入力
   - **テンプレートバージョンの説明**: 任意で説明を入力

### 2.2 起動テンプレートの詳細設定
#### アプリケーションおよびOSイメージ
- **Amazon マシンイメージ (AMI)**: 起動設定で使用していたAMI IDを入力

#### インスタンスタイプ
- **インスタンスタイプ**: 起動設定と同じタイプを選択

#### キーペア（ログイン）
- **キーペア名**: 起動設定で使用していたキーペアを選択

#### ネットワーク設定
- **セキュリティグループ**: 起動設定で使用していたセキュリティグループを選択
- **パブリック IP を自動割り当て**: 必要に応じて設定

#### ストレージを設定
1. **ストレージを追加**をクリック
2. 起動設定の設定に合わせて以下を設定：
   - デバイス名（例：/dev/xvda）
   - ボリュームタイプ（gp3, gp2など）
   - サイズ（GB）
   - 終了時に削除

#### 高度な詳細
- **IAMインスタンスプロファイル**: 起動設定で使用していたロールを選択
- **詳細監視**: CloudWatch詳細監視の有効/無効を設定
- **ユーザーデータ**: 起動設定のユーザーデータをコピー＆ペースト

#### リソースタグ
1. **タグを追加**をクリック
2. **リソースタイプ**で「インスタンス」を選択
3. 必要なタグを追加（Name、Environment等）

### 2.3 起動テンプレートの作成完了
1. 設定内容を確認
2. **起動テンプレートを作成**ボタンをクリック
3. 作成完了メッセージを確認

## 3. Auto Scaling Groupの更新（画面操作）

### 3.1 ASGの起動テンプレート設定変更
1. **Auto Scaling グループ**画面で対象のASGを選択
2. **編集**ボタンをクリック
3. **起動テンプレート**セクションで：
   - **起動テンプレート**: 新しく作成したテンプレートを選択
   - **バージョン**: 「$Latest」または特定のバージョンを選択
4. その他の設定（グループサイズ、ネットワーク等）を確認
5. **更新**ボタンをクリック

### 3.2 設定変更の確認
1. ASGの詳細画面で**起動テンプレート**が新しいものに変更されていることを確認
2. **アクティビティ履歴**タブで設定変更のログを確認

## 4. 検証・テスト（画面操作）

### 4.1 インスタンス起動テスト
1. ASGの詳細画面で**編集**をクリック
2. **グループサイズ**の**必要な容量**を一時的に1つ増やす
3. **更新**をクリック
4. **アクティビティ履歴**タブで新しいインスタンスの起動状況を監視
5. **インスタンス**タブで新しいインスタンスが正常に起動することを確認

### 4.2 新しいインスタンスの確認
1. **EC2** → **インスタンス**で新しく起動したインスタンスを選択
2. 以下を確認：
   - セキュリティグループが正しく設定されているか
   - IAMロールが正しく設定されているか
   - タグが正しく適用されているか
   - ユーザーデータが正常に実行されているか

### 4.3 元の容量に戻す
テストが完了したら、ASGの容量を元の値に戻す

## 5. ロールバック手順（画面操作）

### 5.1 緊急時のロールバック
問題が発生した場合：
1. ASGの詳細画面で**編集**をクリック
2. **起動設定**セクションで元の起動設定を選択
3. **起動テンプレート**の選択を解除
4. **更新**をクリック

### 5.2 起動テンプレートの削除
問題のある起動テンプレートを削除する場合：
1. **起動テンプレート**画面で対象テンプレートを選択
2. **アクション** → **削除**を選択
3. 削除の確認を行う

## 6. 起動設定の削除（画面操作）

移行完了後、不要になった起動設定を削除：
1. **Auto Scaling** → **起動設定**を選択
2. 対象の起動設定を選択
3. **アクション** → **削除**を選択
4. 削除の確認を行う

※注意: ASGで使用中の起動設定は削除できません

---

## 注意事項

1. **メンテナンスウィンドウの確保**: 移行作業は可能な限りメンテナンスウィンドウ内で実施
2. **段階的移行**: 本番環境では段階的に移行を実施（一部のAZから開始）
3. **モニタリング**: 移行後は一定期間、インスタンスの動作を監視
4. **バックアップ**: 移行前に既存設定のバックアップを取得
5. **文書化**: 移行作業の記録と設定変更の文書化

## チェックリスト

- [ ] 既存起動設定の情報収集完了
- [ ] 起動テンプレート作成完了
- [ ] テスト環境での検証完了
- [ ] Auto Scaling Group更新完了
- [ ] 新インスタンス起動テスト完了
- [ ] モニタリング設定完了
- [ ] 古い起動設定の削除（必要に応じて）
- [ ] 文書化完了