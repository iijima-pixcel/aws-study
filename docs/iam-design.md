# IAM リソーススコープ設計方針
本プロジェクトでは、CloudFormation 実行ロールに対して 最小権限の原則（Least Privilege） に基づき、
IAM ポリシーのリソーススコープを段階的に最小化する設計を採用しています。  

## Create 系 API は Resource: "*" にしている理由（重要）
EC2 / ELB / RDS などのリソースは 作成前に ARN が存在しないため、
AWS 公式ベストプラクティスに従い、以下のように Resource: "*" を使用しています。  
- ec2:CreateVpc
- ec2:CreateSubnet
- ec2:CreateSecurityGroup
- elasticloadbalancing:CreateLoadBalancer
- rds:CreateDBInstance  
  など　　  
  （ARN を絞れないため）　
  　
## Modify / Delete 系はタグベース (Project=AwsStudy) で制御
作成済みリソースは CloudFormation によりタグ付与されているため、
以下のようにタグ条件を用いて、削除権限を AwsStudy 関連リソースのみに限定しています。  
```
Condition:
  StringEquals:
    ec2:ResourceTag/Project: AwsStudy
```
これにより CloudFormation が管理するリソースのみ削除でき、
他のプロジェクトへの影響を防止できます。  

## SSM SecureString は対象パラメータ ARN のみに限定  
iam-role.yml では以下のように 1 個のパラメータのみ参照可能 にしています：
```
- Sid: SsmAwsStudyOnly
  Action:
    - ssm:GetParameter
    - ssm:GetParameters
  Resource: !Ref SsmDbPasswordArn
```
* RDS パスワード以外のパラメータは取得不可  
* 最小権限

## KMS decrypt は“SSM 経由のみ” 許可  
```
- Sid: KmsDecryptViaSsm
  Action: kms:Decrypt
  Resource: "*"
  Condition:
    StringEquals:
      kms:ViaService: ssm.ap-northeast-1.amazonaws.com
```
 aws/ssm（AWS Managed Key）を使用  
 SSM 経由での復号のみ許可されるため安全性が高い  
 CMK のキー・ポリシー変更は不要

## 非タグ対応リソースは例外 Statement で個別に許可
一部 AWS リソースは タグ付けやタグ条件による制御に対応していません。  
例：　　
- Listener
- DB Subnet Group
- 一部の ENI
- RDS 自動スナップショット  
  など  
これらはタグ条件で制御できないため、
必要な操作を個別の Statement で許可し、
過剰な権限付与を防いでいます。
# EC2用 IAM ロール（AwsStudy-EC2-Role）について
本プロジェクトでは、CloudFormationExecutionRole の中で
以下の PassRole が定義されています。  
```
- Sid: PassRoleAwsStudy
  Effect: Allow
  Action: iam:PassRole
  Resource: arn:aws:iam::<AWSアカウントID>:role/AwsStudy-EC2-Role
```
これは CloudFormation が EC2 インスタンスを作成する際に、
EC2 用 IAM ロール（AwsStudy-EC2-Role）を EC2 にアタッチする ためのものです。  
## （重要）現時点では AwsStudy-EC2-Role は未作成
本プロジェクトでは、EC2 用の IAM ロールは
今後 App 層構築時に追加する予定のリソースです。  
例：EC2 ロールの典型例（参考）  
```
Resources:
  AwsStudyEc2Role:
    Type: AWS::IAM::Role
    Properties:
      RoleName: AwsStudy-EC2-Role
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
        - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

  AwsStudyEc2InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref AwsStudyEc2Role
```

# 今後の最小権限化方針
本 IAM ロールでは現時点で十分に最小化されていますが、
以下を継続的に改善する方針としています：  
- 新しく追加されるリソースの ARN を段階的に限定
- タグ対応の拡張（ELB / RDS の追加タグ活用）
- 不要 API の削除
- CloudTrail での権限使用状況のモニタリング  
本ロールの最小権限化は段階的に行い、CloudTrail や Stack Events で実際に使用された API を確認しながら不要な権限を随時削除する運用方針としています。
