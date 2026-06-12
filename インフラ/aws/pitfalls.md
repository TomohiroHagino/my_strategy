# 実務でハマる罠まとめ（Pitfalls）（AWS）

## ひとことで言うと
各サービスの「ハマりどころ」を、**コスト・セキュリティ**を軸に集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、事故った時の原因切り分けの入口として使う。

## 役割・なぜ必要か
- AWSは「ちょっとした設定ミス」が **情報漏洩** や **高額請求** に直結する。症状から該当箇所へ素早く飛ぶための索引。
- 特に**セキュリティ事故**と**課金事故**は初心者がやらかしがちな2大ジャンル。ここを一望できると事前に潰せる。

## 権限 / アカウント（IAM）
- **ルートユーザー常用**：ルートは全権限で事故時の被害が最大。日常作業は**IAMユーザー/ロール**で行い、ルートはMFA必須・封印する。→ [iam.md](./iam.md)
- **過剰権限（最小権限違反）**：`*:*` や `AdministratorAccess` を雑に付与しがち。**必要な操作だけ**に絞り、ロールで一時権限を渡す。→ [iam.md](./iam.md)
- **アクセスキーをGitにコミット**：`AKIA...` を誤コミット→ボットが秒で拾い不正利用＆高額請求。鍵は環境変数/Secrets Managerへ、即ローテーション、`git-secrets`等で予防。→ [iam.md](./iam.md)

## ストレージ / 公開設定（S3）
- **S3バケット誤公開で情報漏洩**：ACL/バケットポリシーの設定ミスで全世界に公開→個人情報流出。**Block Public Access を全ON**を既定にし、公開は意図した範囲だけ。→ [s3.md](./s3.md)

## ネットワーク / 公開範囲（VPC・EC2）
- **セキュリティグループ `0.0.0.0/0` 全開放**：SSH(22)/RDP(3389)/DBポートを全世界に開けると総当たり・侵入の標的。**自IP/社内CIDRに限定**、踏み台やSSMで代替。→ [ec2.md](./ec2.md) / [vpc.md](./vpc.md)
- **NATゲートウェイ課金**：プライベートサブネットの外向き通信用NAT GWは**起動しているだけで時間課金＋データ処理課金**。不要なら削除、S3/DynamoDBは**VPCエンドポイント**でNATを通さず節約。→ [vpc.md](./vpc.md)

## コスト / 課金事故
- **課金アラート未設定**：気づいたら高額請求。**Budgets / 課金アラーム**を最初に設定し、想定額超過で通知。→ [getting_started.md](./getting_started.md)
- **インスタンス停止忘れ**：検証用EC2/RDSを起動しっぱなしで課金が積もる。使わないものは**停止/削除**、タグ＋自動停止で管理。→ [ec2.md](./ec2.md)
- **ログ保存料金の放置**：CloudWatch Logs の保持期間が既定で無期限→ストレージ課金が膨張。**保持期間を必ず設定**。→ [monitoring.md](./monitoring.md)

## サーバーレス / DB設計
- **Lambdaのタイムアウト/コールドスタート**：既定タイムアウトは短く処理途中で打ち切り。VPC接続や初回起動の**コールドスタート**で遅延。タイムアウト/メモリを適正化、初期化処理は外出し。→ [lambda.md](./lambda.md)
- **DynamoDBはキー設計前提**：RDB感覚で後からクエリ追加は不可。**アクセスパターンを先に決めてパーティションキー/GSIを設計**。スキャン多用は遅く高コスト。→ [dynamodb.md](./dynamodb.md)

## 配信 / DNS / IaC
- **CloudFront証明書はus-east-1**：CloudFrontに紐づけるACM証明書は**バージニア北部（us-east-1）でのみ**発行/参照可能。他リージョンで作ると紐づかない。→ [cloudfront_route53.md](./cloudfront_route53.md)
- **IaCのドリフト（drift）**：コンソールで手動変更するとコード（CloudFormation/Terraform）の定義と実体がズレる。**変更はコード経由**に統一し、`drift detection` / `terraform plan` で検知。→ [iac.md](./iac.md)

## 関連
[iam.md](./iam.md) / [s3.md](./s3.md) / [ec2.md](./ec2.md) / [vpc.md](./vpc.md) / [getting_started.md](./getting_started.md) / [lambda.md](./lambda.md) / [dynamodb.md](./dynamodb.md) / [cloudfront_route53.md](./cloudfront_route53.md) / [iac.md](./iac.md) / [monitoring.md](./monitoring.md)
