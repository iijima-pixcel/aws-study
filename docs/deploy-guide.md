### デプロイ手順
### 🧾 前提  
-  このテンプレート群はIAMロールを作成・利用するため、
すべてのデプロイコマンドに --capabilities CAPABILITY_NAMED_IAM が必須 です。  
指定しない場合、次のようなエラーで失敗します
```
An error occurred (InsufficientCapabilitiesException)
when calling the CreateStack operation:
Requires capabilities : [CAPABILITY_NAMED_IAM]
```
### 🪪 まず最初に実行：CloudFormation 実行ロールのデプロイ
>　　IAMロールを作成しないと、後続のNetwork/Security/Appスタックは実行できません。
```bash
# Iam-Roleスタック
aws cloudformation deploy --template-file AWS-CloudFormation/iam-role.yml --stack-name CloudFormationExecutionRoleStack --capabilities CAPABILITY_NAMED_IAM --region ap-northeast-1
```
# デプロイ前の事前チェック
デプロイ前には必ず以下を実行してください。
## Template の構文チェック（validate-template）
```
aws cloudformation validate-template \
  --template-body file://AWS-CloudFormation/App.yml
```
 正常な例
```
Description: Aws-Study-ApplicationLayer
Parameters:
- Description: Name of the key pair to use for the EC2 instance
  NoEcho: false
  ParameterKey: KeyName
- DefaultValue: admin
  Description: Master username for the RDS instance
  NoEcho: false
  ParameterKey: DBMasterUsername
- Description: AMI ID to use for the EC2 instance
  NoEcho: false
  ParameterKey: AMI
```

## changeSet を使った安全なデプロイフロー
  本番デプロイ前には、必ず ChangeSet を作成し,  
  CloudFormation が実行する予定の API（Create / Modify / Replace）を事前に確認してください。
  ### ChangeSet（初回作成 CREATE ChangeSet）
  ```
  aws cloudformation create-change-set \
  --stack-name AwsStudy-App-stack \
  --change-set-name PreDeployCheck \
  --change-set-type CREATE \
  --template-body file://AWS-CloudFormation/App.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::<AWSアカウントID>:role/CloudFormationExecutionRole \
  --parameters \
      ParameterKey=KeyName,ParameterValue=<YOUR-KEYPAIR-NAME> \
      ParameterKey=AMI,ParameterValue=<YOUR-AMI-ID> \
      ParameterKey=DBMasterUsername,ParameterValue=<YOUR-DB-USERNAME> \
  --region ap-northeast-1
  ``` 
   ここも <AWSアカウントID> を必ず置き換えてください。
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

  ### スタックイベント確認
  ```
  aws cloudformation describe-stack-events \
  --stack-name AwsStudy-App \
  --region ap-northeast-1 \
  --query "StackEvents[?ResourceStatusReason!=null].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]" \
  --output text
  ```

# ChangeSet 運用フローチャート（簡易）
1. ChaChangeSet 作成
2. ChangeSet の内容（Add/Modify/Replace）を確認
3. IAM 権限不足があれば修正
4. 問題なければ ChangeSet を実行（apply）
5. スタックイベントで進捗確認
6. 完了（CREATE_COMPLETE）

#   デプロイ手順（正しい順番）
### Network スタック
```
aws cloudformation deploy \
  --template-file AWS-CloudFormation/Network.yml \
  --stack-name AwsStudy-Network-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::<AWSアカウントID>:role/CloudFormationExecutionRole
```  
 ここも <AWSアカウントID> を必ず置き換えてください。
### Security スタック
```
aws cloudformation deploy \
  --template-file AWS-CloudFormation/Security.yml \
  --stack-name AwsStudy-Security-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::<AWSアカウントID>:role/CloudFormationExecutionRole \
  --parameter-overrides \
      CidrIpFromInternet=<YOUR-CIDR-IP>   # ここを自分のアクセス元 IP/CIDR に置き換える
```

### App スタック
```
aws cloudformation deploy \
  --template-file AWS-CloudFormation/App.yml \
  --stack-name AwsStudy-App-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::<AWSアカウントID>:role/CloudFormationExecutionRole \
  --parameter-overrides \
      KeyName=<YOUR-KEYPAIR-NAME> \        # ここを自分の EC2 キーペア名に置き換える
      AMI=<YOUR-AMI-ID> \                  # ここを対象リージョンの AMI ID に置き換える
      DBMasterUsername=<YOUR-DB-USERNAME>  # ここを RDS マスターユーザー名に置き換える
```
注意  
KeyName: AWS EC2 のキーペア名  
AMI: 対象リージョンの有効な AMI ID  
DBMasterUsername: RDS のマスターユーザー名

##  KMSキーについて
- 本環境では、SSM パラメータストアの SecureString 暗号化にAWSマネージドキー (`aws/ssm`) を使用しています。
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
 ここも必ず <AWSアカウントID> を置き換えてください。
 
## デプロイ順序
1. AWS-CloudFormation/iam-role.ymlで CloudFormation 実行ロールを作成
2. AWS-CloudFormation/Network.ymlで基盤ネットワーク構築
3. AWS-CloudFormation/Security.ymlでセキュリティグループ等構築
4. AWS-CloudFormation/App.ymlでEC2 / RDS などアプリ層構築  
 
 注意  :iam-role.yml を最初に作成しないと、後続スタックがロールを参照できずエラーになります。
