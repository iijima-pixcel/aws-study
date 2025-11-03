##  CloudFormation 実行ロールの指定方法
本リポジトリは、Network / Security / App の3層構成でデプロイします。  
各スタック作成時に、CloudFormation 実行ロールを明示的に指定する必要があります。

### 実行ロール情報
- ロール名: `CloudFormationExecutionRole`
- ARN: `arn:aws:iam::<AWSアカウントID>:role/CloudFormationExecutionRole`
  
- **Role ARN の確認方法**
> IAM ロール作成後に以下のコマンドで ARN を確認し、上記の `<AWSアカウントID>` のところを、使用者の`<AWSアカウントID>`に置き換えてください
> ```bash
> aws iam get-role --role-name CloudFormationExecutionRole --query "Role.Arn" --output text
> ```

### デプロイ手順
```bash
# Network スタック
aws cloudformation deploy \
  --template-file AWS-CloudFormation/Network.yml \
  --stack-name network-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::<AWSアカウントID>:role/CloudFormationExecutionRole

# Security スタック
aws cloudformation deploy \
  --template-file AWS-CloudFormation/Security.yml \
  --stack-name security-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::<AWSアカウントID>:role/CloudFormationExecutionRole \
  --parameter-overrides NetworkStackName=network-stack

# App スタック
aws cloudformation deploy \
  --template-file AWS-CloudFormation/App.yml \
  --stack-name app-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::<AWSアカウントID>:role/CloudFormationExecutionRole \
  --parameter-overrides \
    NetworkStackName=network-stack 
```

##  KMSキーについて
- 本環境では、SSM パラメータストアの SecureString 暗号化にAWSマネージドキー (`aws/ssm`)** を使用しています。
- このキーは AWS によって管理されており、キー・ポリシーの直接編集は行えません。
- CloudFormation や SSM からの復号 (kms:Decrypt) は、AWS によるサービスロール経由で自動的に許可されます。
- そのため、追加のキー・ポリシー設定は不要です。
- カスタム CMK を使う場合はキー・ポリシーで当該実行ロールを許可する必要があります。
  - キーのポリシー内で、CloudFormation 実行ロールを許可してください
  - 許可例
  ```json
  {
  "Sid": "Allow use of the key for CloudFormationExecutionRole",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::<自分のAWSアカウントID>:role/CloudFormationExecutionRole"
  },
  "Action": [
    "kms:Decrypt",
    "kms:Encrypt",
    "kms:GenerateDataKey*"
  ],
  "Resource": "*"
  }

## デプロイ順序
1. AWS-CloudFormation/iam-role.ymlで CloudFormation 実行ロールを作成
2. AWS-CloudFormation/Network.ymlで基盤ネットワーク構築
3. AWS-CloudFormation/Security.ymlでセキュリティグループ等構築
4. AWS-CloudFormation/App.ymlでEC2 / RDS などアプリ層構築

## （CI/CD利用時の補足）
- CI/CD 実行ロールがこのロールを引き受ける方式ではなく、
スタック作成時に --role-arn を指定して実行する方式を採用しています。

## デプロイ実行主体
本スタックは以下のロールを使用してデプロイします。
- 実行ロール: `arn:aws:iam::205619292566:role/CloudFormationExecutionRole`
- 実行ユーザー: IAM 管理者ユーザー（手動デプロイ時）
## 想定される失敗時の確認箇所
- **CloudFormation Stack Events**: 各リソースの作成／更新エラーを確認／権限の許可
- **CloudTrail**: 権限不足や API 呼び出し拒否の発生履歴を確認
- **KMS**: 暗号化リソース（例: SSM SecureString, RDS 暗号化）の復号権限エラーを確認
