# AWS 実務リファレンス（索引）

> **Amazon Web Services** を、アプリ開発者が実際に使うコアサービス中心に項目別でまとめる。
> 各ファイルは「とは → 役割・なぜ → 基本の使い方（CLI/コンソール）→ 実務の勘所 → ハマり所 → 関連」。

## まず押さえる3本柱
```
 ① IAM（誰が何をできるか）= 最小権限    … 全ての土台。事故もここから
 ② コンピュート（EC2 / Lambda / ECS）   … どこでコードを動かすか
 ③ ストレージ/DB（S3 / RDS / DynamoDB）  … データをどこに置くか
```

## 項目（各ファイルへ）

### 手を動かす（実習）
- [ハンズオン.md](./ハンズオン.md) … **LocalStackで課金ゼロ**：S3バケット作成→put/get→DynamoDB→失敗と修正（0から動く）

### 基礎 / 権限
- [getting_started.md](./getting_started.md) … 始め方（アカウント / リージョン / CLI / 課金アラート）
- [iam.md](./iam.md) … IAM（ユーザー / ロール / ポリシー / 最小権限）とは

### コンピュート
- [ec2.md](./ec2.md) … EC2（仮想サーバー）とは
- [lambda.md](./lambda.md) … Lambda（サーバーレス関数）とは
- [containers.md](./containers.md) … コンテナ全体像（ECS / Fargate / EKS / ECR の俯瞰）とは
- [ecs.md](./ecs.md) … ECS（コンテナ実行：クラスタ / タスク / サービス / Fargate）とは
- [ecr.md](./ecr.md) … ECR（コンテナイメージのレジストリ）とは

### ストレージ / DB
- [s3.md](./s3.md) … S3（オブジェクトストレージ）とは
- [rds.md](./rds.md) … RDS / Aurora（マネージドRDB）とは
- [dynamodb.md](./dynamodb.md) … DynamoDB（マネージドNoSQL）とは

### ネットワーク / 配信 / 運用
- [vpc.md](./vpc.md) … VPC（仮想ネットワーク / セキュリティグループ）とは
- [api_gateway.md](./api_gateway.md) … API Gateway（APIの入口）とは
- [cloudfront_route53.md](./cloudfront_route53.md) … CloudFront（CDN）/ Route53（DNS）とは
- [iac.md](./iac.md) … IaC（CloudFormation / CDK / Terraform）とは
- [monitoring.md](./monitoring.md) … CloudWatch（監視 / ログ / アラーム）とは
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ（コスト・セキュリティ）

## 各ファイルの書式（テンプレ）
```markdown
# {サービス名}（AWS）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の使い方（CLI / コンソール）
## 実務での勘所
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
