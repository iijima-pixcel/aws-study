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
  --stack-name AwsStudy-Network-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::<AWSアカウントID>:role/CloudFormationExecutionRole

# Security スタック
aws cloudformation deploy \
  --template-file AWS-CloudFormation/Security.yml \
  --stack-name AwsStudy-Security-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::<AWSアカウントID>:role/CloudFormationExecutionRole \
  --parameter-overrides 
      CidrIpFromInternet=<YOUR-CIDR-IP> \  # ここを自分のアクセス元 IP/CIDR に置き換える
⚠️注意
<YOUR-CIDR-IP> はアクセスを許可する IP/CIDR を指定します（例: 36.8.0.45/32）。

# App スタック
aws cloudformation deploy \
  --template-file AWS-CloudFormation/App.yml \
  --stack-name AwsStudy-App-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::<AWSアカウントID>:role/CloudFormationExecutionRole \
  --parameter-overrides \
      KeyName=<YOUR-KEYPAIR-NAME> \        # ここを自分の EC2 キーペア名に置き換える
      AMI=<YOUR-AMI-ID> \                  # ここを対象リージョンの AMI ID に置き換える
      DBMasterUsername=<YOUR-DB-USERNAME>  # ここを RDS マスターユーザー名に置き換える
⚠️注意
KeyName: AWS EC2 のキーペア名
AMI: 対象リージョンの有効な AMI ID
DBMasterUsername: RDS のマスターユーザー名
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

## 事前確認（ChangeSet とテストデプロイ）
  本番デプロイ前に ChangeSet を使って作成内容を確認してください。
  ### ChangeSet 作成例（App スタック）
  ```
  aws cloudformation create-change-set \
  --stack-name AwsStudy-App-stack \
  --change-set-name PreDeployCheck \
  --template-body file://AWS-CloudFormation/App.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::<AWSアカウントID>:role/CloudFormationExecutionRole \
  --parameters ParameterKey=KeyName,ParameterValue=<YOUR-KEYPAIR-NAME> \
               ParameterKey=AMI,ParameterValue=<YOUR-AMI-ID> \
               ParameterKey=DBMasterUsername,ParameterValue=<YOUR-DB-USERNAME> \             
  --region ap-northeast-1
  ``` 
  ### ChangeSet 内容確認
  ```
  aws cloudformation describe-change-set \
  --stack-name AwsStudy-App-stack \
  --change-set-name PreDeployCheck \
  --region ap-northeast-1
  ```
  ### ChangeSet 実行
  ```
  aws cloudformation execute-change-set \
  --stack-name AwsStudy-App-stack \
  --change-set-name PreDeployCheck \
  --region ap-northeast-1
  ```
⚠️備考:  
ChangeSet を事前に作成して確認することをお勧めします。IAM ロールを作成してからでないと他テンプレートは参照できないため、最初に iam-role.yml を作成してください。
  ### スタックイベント確認
  ```
  aws cloudformation describe-stack-events \
  --stack-name AwsStudy-App \
  --region ap-northeast-1 \
  --query "StackEvents[?ResourceStatusReason!=null].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]" \
  --output text
  ```
  
💡 解説
- --query  
  → 失敗や警告など、理由が付与されたイベントのみを抽出します。
- --output text  
→ 見やすい一行形式で出力します。

✅ チェックポイント
- CREATE_COMPLETE になっていれば成功
- ROLLBACK_IN_PROGRESS や FAILED の場合は、権限や依存関係を確認
- 詳細を CloudFormation コンソールの「Events」タブでも確認可能

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
