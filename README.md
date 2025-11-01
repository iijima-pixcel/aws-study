##  CloudFormation 実行ロールの指定方法
本リポジトリは、Network / Security / App の3層構成でデプロイします。  
各スタック作成時に、CloudFormation 実行ロールを明示的に指定する必要があります。

### 実行ロール情報
- ロール名: `CloudFormationExecutionRole`
- ARN: `arn:aws:iam::205619292566:role/CloudFormationExecutionRole`
  
### デプロイ手順
```bash
# Network スタック
aws cloudformation deploy \
  --template-file AWS-CloudFormation/Network.yml \
  --stack-name network-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::205619292566:role/CloudFormationExecutionRole

# Security スタック
aws cloudformation deploy \
  --template-file AWS-CloudFormation/Security.yml \
  --stack-name security-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::205619292566:role/CloudFormationExecutionRole \
  --parameter-overrides NetworkStackName=network-stack

# App スタック
aws cloudformation deploy \
  --template-file AWS-CloudFormation/App.yml \
  --stack-name app-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::205619292566:role/CloudFormationExecutionRole \
  --parameter-overrides \
    NetworkStackName=network-stack \
```

##  KMSキーについて
- 本環境では、SSM パラメータストアの SecureString 暗号化にAWSマネージドキー (`aws/ssm`)** を使用しています。
- このキーは AWS によって管理されており、キー・ポリシーの直接編集は行えません。
- CloudFormation や SSM からの復号 (kms:Decrypt) は、AWS によるサービスロール経由で自動的に許可されます。
- そのため、追加のキー・ポリシー設定は不要です。
- カスタム CMK を使う場合はキー・ポリシーで当該実行ロールを許可する必要があります。

## （CI/CD利用時の補足）
- CI/CD 実行ロールがこのロールを引き受ける方式ではなく、
スタック作成時に --role-arn を指定して実行する方式を採用しています。
