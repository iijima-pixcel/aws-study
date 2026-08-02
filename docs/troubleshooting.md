# CloudFormation変更セット作成時必ず確認するポイント
  | 確認項目                               | 意味                   |
| ---------------------------------- | -------------------- |
| **Action: Add**                    | 新規作成                 |
| **Action: Modify**                 | 設定変更（非破壊）            |
| **Action: Replace**                |  破壊的変更（再作成が発生）     |
| **AWS::RDS::DBInstance が Replace** |  DB 再作成 → データ消失の危険 |
| **AWS::EC2::Instance が Replace**   |  EC2 作り直し          |
| **AccessDenied**                   | IAM 権限不足。ロール修正が必要    |

#  よくある失敗時の確認ポイント
| 種類       | 典型エラー                              | 確認対象                              |
| -------- | ---------------------------------- | --------------------------------- |
| IAM 権限不足 | AccessDenied                       | CloudFormationExecutionRole のポリシー |
| リソース重複   | AlreadyExists                      | 手動作成リソースを削除                       |
| サブネット誤り  | Subnet does not exist              | Network スタックの Export              |
| RDS 失敗   | DB subnet group does not cover AZs | private subnet の組み合わせ             |
| KMS 権限不足 | AccessDeniedException              | CMK ポリシー                          |
