# GCP 実務リファレンス（索引）

> **Google Cloud Platform** を、アプリ開発者が実際に使うコアサービス中心に項目別でまとめる。
> 各ファイルは「とは → 役割・なぜ → 基本の使い方（gcloud/コンソール）→ 実務の勘所 → ハマり所 → 関連」。

## まず押さえる3本柱
```
 ① IAM ＋ プロジェクト（全リソースは「プロジェクト」に属する）
 ② コンピュート（Compute Engine / Cloud Run / Cloud Functions）
 ③ ストレージ/DB（Cloud Storage / Cloud SQL / Firestore）＋ BigQuery（分析）
```

## 項目（各ファイルへ）

### 基礎 / 権限
- [getting_started.md](./getting_started.md) … 始め方（プロジェクト / gcloud / 課金）
- [iam.md](./iam.md) … IAM（メンバー / ロール / 最小権限）とは

### コンピュート
- [compute_engine.md](./compute_engine.md) … Compute Engine（VM）とは
- [cloud_run.md](./cloud_run.md) … Cloud Run（サーバーレスコンテナ）とは ← GCPの主役
- [cloud_functions.md](./cloud_functions.md) … Cloud Functions（FaaS）とは

### ストレージ / DB / 分析
- [cloud_storage.md](./cloud_storage.md) … Cloud Storage（オブジェクト）とは
- [cloud_sql.md](./cloud_sql.md) … Cloud SQL（マネージドRDB）とは
- [firestore.md](./firestore.md) … Firestore（マネージドNoSQL）とは
- [bigquery.md](./bigquery.md) … BigQuery（データウェアハウス）とは ← GCPの目玉

### ネットワーク / 運用
- [vpc.md](./vpc.md) … VPC（仮想ネットワーク / ファイアウォール）とは
- [iac.md](./iac.md) … IaC（Terraform / Deployment Manager）とは
- [monitoring.md](./monitoring.md) … Cloud Monitoring / Logging とは
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ（コスト・セキュリティ）

## 各ファイルの書式（テンプレ）
```markdown
# {サービス名}（GCP）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の使い方（gcloud / コンソール）
## 実務での勘所
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
