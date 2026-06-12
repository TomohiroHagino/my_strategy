# IAM（AWS）

## ひとことで言うと
**Identity and Access Management** ＝「**誰が（認証）・何に・何をできるか（認可）**」を管理するAWSの土台サービス。全AWSサービスへのアクセスはIAMの許可を通る。

## 役割・なぜ必要か
- AWSは「誰でも何でもできる」状態だと事故が起きる。IAMで**操作を許可・拒否**することで、開発者・サーバー・外部システムそれぞれに必要最小限の権限だけ渡せる。
- セキュリティ事故の多くは「権限の渡しすぎ」「鍵の漏洩」から起きる。IAMを正しく設計することが、AWSを安全に使う前提条件になる。
- 4つの登場人物を押さえると全体が見える：
  - **ユーザー（User）**：人。ログインして操作する主体。
  - **グループ（Group）**：ユーザーの束。権限をまとめて付与する単位（例：developers グループ）。
  - **ロール（Role）**：一時的に「成り代わる」権限の入れ物。**人ではなくAWSサービス（EC2やLambda）に付与**して、鍵を持たせずに権限を渡せる。
  - **ポリシー（Policy）**：許可/拒否を書いた**JSONのルール**。上記に貼り付けて効力を持つ。

## 基本の使い方（CLI / コンソール）
ポリシーはJSONで「どのアクションを・どのリソースに・許可/拒否するか」を書く：
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    }
  ]
}
```
このポリシーは「my-app-bucket の中身を読み書きしてよい。それ以外は不可」を意味する（最小権限）。

```bash
# 自分が誰かを確認
aws sts get-caller-identity

# ユーザー作成 → グループに入れる（権限はグループ側に付ける）
aws iam create-user --user-name alice
aws iam create-group --group-name developers
aws iam add-user-to-group --user-name alice --group-name developers

# グループにマネージドポリシーを付与（例：S3読み取り専用）
aws iam attach-group-policy --group-name developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# ロール作成（EC2に成り代わらせる例。Trust Policyで「誰が引き受けられるか」を定義）
aws iam create-role --role-name ec2-s3-role \
  --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name ec2-s3-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

コンソールでは IAM → Users / Groups / Roles / Policies から同じことができる。MFAは IAM → ユーザー → セキュリティ認証情報 から各ユーザーに設定。

## 実務での勘所
- **最小権限の原則（least privilege）**：必要な操作だけ許可し、後から足りなければ追加する。「とりあえず全許可」は禁物。
- **権限はグループに付ける**：個々のユーザーに直接ポリシーを貼ると管理不能になる。developers / admins などグループ単位で運用する。
- **サーバーにはロールを使う（鍵を置かない）**：EC2やLambdaにロールを付与すれば、アクセスキーを発行せずにS3等へアクセスできる。**ローカルにキーを置かない＝漏洩しない**。これが鉄則。
- **MFAを全員に**：パスワード漏洩だけでは入れないようにする。
- アクセスキー（CLI/SDK用の長期鍵）は人間のローカル開発でだけ使い、サーバーでは使わない。発行したら定期的にローテーション。
- マネージドポリシー（AWS提供の既製ポリシー）で足りなければ、カスタムポリシーを書く。`*ReadOnlyAccess` 系は安全側で便利。

## ハマりどころ / アンチパターン
- **`AdministratorAccess` の乱用**：面倒だからと全員に管理者権限を付ける。1人漏洩すれば全損。本当に必要な少数だけに限定する。
- **アクセスキーのgitコミット漏洩**：`AKIA...` をソースに直書き→GitHub公開で数分でbotに拾われ、不正にEC2を大量起動されマイニングに悪用、高額請求。**キーはコードに書かず、サーバーはロール、ローカルは`~/.aws/credentials`**。→ [getting_started.md](./getting_started.md)
- **ルートユーザーで日常作業**：ルートは全権限かつ無効化不可。封印してIAMユーザーを使う。
- **ポリシーの`Resource: "*"`乱用**：「全リソースに対して全アクション」は実質Admin。対象を絞る。
- **明示的Denyの見落とし**：IAMは「Denyが最優先」。Allowを足したのに動かない時、別のポリシーでDenyされていることがある。

## 関連
[getting_started.md](./getting_started.md)
