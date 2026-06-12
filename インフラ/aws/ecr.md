# ECR（Elastic Container Registry）（AWS）

## ひとことで言うと
AWSマネージドの**コンテナイメージ置き場（レジストリ）**。Docker Hub のプライベート自社版。ビルドしたイメージを push し、ECS / EKS / Lambda がここから pull して起動する。

## 役割・なぜ必要か
- イメージの**保管・バージョン（タグ）管理・配布**の中心。`docker build` した成果物の置き場。
- プライベートで **IAM で権限管理**。**脆弱性スキャン**・**ライフサイクルポリシー（自動削除）**が内蔵。
- ECS/EKS は「どのイメージを動かすか」をECRのURIで指定する。→ [ecs.md](./ecs.md)

## 基本の使い方（CLI）
```bash
ACCOUNT=123456789012; REGION=ap-northeast-1; REPO=myapp
# 1) リポジトリ作成（イメージスキャン有効・タグ上書き禁止）
aws ecr create-repository --repository-name $REPO \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability IMMUTABLE

# 2) Docker を ECR にログイン（tokenは約12時間で失効。CIでは都度実行）
aws ecr get-login-password --region $REGION \
  | docker login --username AWS --password-stdin $ACCOUNT.dkr.ecr.$REGION.amazonaws.com

# 3) ビルド → タグ付け → push（タグはコミットハッシュで固定）
TAG=$(git rev-parse --short HEAD)
docker build -t $REPO:$TAG .
docker tag $REPO:$TAG $ACCOUNT.dkr.ecr.$REGION.amazonaws.com/$REPO:$TAG
docker push $ACCOUNT.dkr.ecr.$REGION.amazonaws.com/$REPO:$TAG
```
ライフサイクルポリシー（古いイメージを自動削除）：
```json
{ "rules": [{
  "rulePriority": 1,
  "description": "直近10個だけ残す",
  "selection": { "tagStatus": "any", "countType": "imageCountMoreThan", "countNumber": 10 },
  "action": { "type": "expire" }
}]}
```

## 実務の勘所
- **タグは `latest` 固定にしない**：コミットハッシュ / semver で固定 → どのコミットが本番か明確、ロールバックも容易。→ [ecs.md](./ecs.md)
- **`IMMUTABLE` タグ**：同じタグの上書きを禁止し、「いつの間にか中身が変わる」事故を防ぐ。
- **ライフサイクルポリシー必須**：放置するとイメージが溜まり**容量＝コスト**が増える。世代数や日数で自動削除。
- **イメージスキャン**：`scanOnPush` で push時に脆弱性検査（拡張スキャンは Amazon Inspector 連携）。→ [../../DevOps/automation_quality.md](../../DevOps/automation_quality.md)
- **権限**：ECS の executionRole に pull 権限、CI に push 権限（最小権限）。→ [iam.md](./iam.md)
- **クロスアカウント/リージョン**：リポジトリポリシーで共有、**レプリケーション**で別リージョンへ複製。
- **pull through cache**：Docker Hub 等の公開イメージを ECR 経由でキャッシュ（レート制限回避・高速化）。

## ハマりどころ / アンチパターン
- **プライベートサブネットから pull できない**：NAT Gateway か **VPCエンドポイント（ecr.api / ecr.dkr / s3）** が無いと、ECS のタスク起動が失敗する。→ [vpc.md](./vpc.md)
- **ログイン token の失効**：`get-login-password` のtokenは約12時間。CIでは push の直前に毎回ログインする。
- **`latest` 運用**：どのコミットが本番か分からず、ロールバック不能。タグ固定で回避。
- **ライフサイクルポリシー無し**：イメージが無限に溜まりストレージ課金が膨らむ。
- **リージョン違い**：別リージョンのECRは別物。ECSと同じリージョンに置くか、レプリケーションする。

## 関連
[ecs.md](./ecs.md) / [containers.md](./containers.md) / [iam.md](./iam.md) / [vpc.md](./vpc.md)
コンテナ自体は [../docker/](../docker/) ／ IaC実例 [../terraform/example_aws_ecs.md](../terraform/example_aws_ecs.md)
