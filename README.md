# CloudFormation 実行ロールの指定方法
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
✅ 正常な例
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
  ### ChangeSet 内容確認
  ```
  aws cloudformation describe-change-set \
  --stack-name AwsStudy-App-stack \
  --change-set-name PreDeployCheck \
  --region ap-northeast-1
  ```
  ### ここで必ず確認するポイント
  | 確認項目                               | 意味                   |
| ---------------------------------- | -------------------- |
| **Action: Add**                    | 新規作成                 |
| **Action: Modify**                 | 設定変更（非破壊）            |
| **Action: Replace**                | ⚠️ 破壊的変更（再作成が発生）     |
| **AWS::RDS::DBInstance が Replace** | ⚠️ DB 再作成 → データ消失の危険 |
| **AWS::EC2::Instance が Replace**   | ⚠️ EC2 作り直し          |
| **AccessDenied**                   | IAM 権限不足。ロール修正が必要    |

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
  
💡 解説
- --query  
  → 失敗や警告など、理由が付与されたイベントのみを抽出します。
- --output text  
→ 見やすい一行形式で出力します。

# ✅ よくある失敗時の確認ポイント
| 種類       | 典型エラー                              | 確認対象                              |
| -------- | ---------------------------------- | --------------------------------- |
| IAM 権限不足 | AccessDenied                       | CloudFormationExecutionRole のポリシー |
| リソース重複   | AlreadyExists                      | 手動作成リソースを削除                       |
| サブネット誤り  | Subnet does not exist              | Network スタックの Export              |
| RDS 失敗   | DB subnet group does not cover AZs | private subnet の組み合わせ             |
| KMS 権限不足 | AccessDeniedException              | CMK ポリシー                          |

# ChangeSet 運用フローチャート（簡易）
1. ChaChangeSet 作成
2. ChangeSet の内容（Add/Modify/Replace）を確認
3. IAM 権限不足があれば修正
4. 問題なければ ChangeSet を実行（apply）
5. スタックイベントで進捗確認
6. 完了（CREATE_COMPLETE）
 
#  ✅ デプロイ手順（正しい順番）
### Network スタック
```
aws cloudformation deploy \
  --template-file AWS-CloudFormation/Network.yml \
  --stack-name AwsStudy-Network-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn arn:aws:iam::<AWSアカウントID>:role/CloudFormationExecutionRole
```
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
⚠️注意
<YOUR-CIDR-IP> はアクセスを許可する IP/CIDR を指定します（例: 36.8.0.45/32）。

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
⚠️注意
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

## デプロイ順序
1. AWS-CloudFormation/iam-role.ymlで CloudFormation 実行ロールを作成
2. AWS-CloudFormation/Network.ymlで基盤ネットワーク構築
3. AWS-CloudFormation/Security.ymlでセキュリティグループ等構築
4. AWS-CloudFormation/App.ymlでEC2 / RDS などアプリ層構築  
 
⚠️ 注意:iam-role.yml を最初に作成しないと、後続スタックがロールを参照できずエラーになります。
  
## （CI/CD利用時の補足）
- CI/CD 実行ロールがこのロールを引き受ける方式ではなく、
スタック作成時に --role-arn を指定して実行する方式を採用しています。

## デプロイ実行主体
本スタックは以下のロールを使用してデプロイします。
- 実行ロール: `arn:aws:iam::<AWSアカウントID>:role/CloudFormationExecutionRole"　　

⚠️ 注意: 使用者のアカウントIDに入れ替えてください
- 実行ユーザー: IAM 管理者ユーザー（手動デプロイ時）
## 想定される失敗時の確認箇所
- **CloudFormation Stack Events**: 各リソースの作成／更新エラーを確認／権限の許可
- **CloudTrail**: 権限不足や API 呼び出し拒否の発生履歴を確認
- **KMS**: 暗号化リソース（例: SSM SecureString, RDS 暗号化）の復号権限エラーを確認
