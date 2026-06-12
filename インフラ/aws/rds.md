# RDS / Aurora（AWS）

## ひとことで言うと
**マネージドなリレーショナルDB（RDB）**。MySQL / PostgreSQL / MariaDB / Oracle / SQL Server などのDBエンジンを、サーバー構築・パッチ適用・バックアップを **AWSに任せた状態** で使えるサービス。**Aurora** はAWSが作り直したMySQL/PostgreSQL互換の高性能版。

## 役割・なぜ必要か
- 自前でEC2にMySQLを入れると「OSパッチ・DBバージョンアップ・バックアップ・冗長化・監視」を全部自分でやることになる。RDSは**この運用作業を肩代わり**してくれる。
- アプリ開発者は「接続先（エンドポイント）・ユーザー・パスワード」を知っていればよく、**DBの面倒を見る時間をアプリに回せる**のが本質。
- 主な機能：
  - **マルチAZ**：別のアベイラビリティゾーンに**待機系（スタンバイ）**を持ち、障害時に自動フェイルオーバー（= 冗長化）。
  - **リードレプリカ**：読み取り専用の複製を増やし、**参照が多いアプリの負荷を分散**。
  - **自動バックアップ / スナップショット**：日次バックアップ＋トランザクションログで**任意時点に復元（PITR）**できる。

## 基本の使い方（CLI / コンソール）
```bash
# DBインスタンス作成（PostgreSQL / マルチAZ / プライベート想定）
aws rds create-db-instance \
  --db-instance-identifier myapp-prod \
  --engine postgres \
  --engine-version 16.3 \
  --db-instance-class db.t4g.micro \
  --allocated-storage 20 \
  --master-username appuser \
  --manage-master-user-password \
  --no-publicly-accessible \
  --vpc-security-group-ids sg-0123456789abcdef0 \
  --db-subnet-group-name myapp-private-subnets \
  --backup-retention-period 7

# 接続先（エンドポイント）を確認
aws rds describe-db-instances \
  --db-instance-identifier myapp-prod \
  --query 'DBInstances[0].Endpoint.Address' --output text

# リードレプリカを作る（参照分散）
aws rds create-db-instance-read-replica \
  --db-instance-identifier myapp-prod-ro \
  --source-db-instance-identifier myapp-prod
```
```bash
# 復元はスナップショットから「新しいインスタンス」として作る
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier myapp-restore \
  --db-snapshot-identifier myapp-prod-snapshot-2026-06-11
```
- コンソール：RDS → 「データベースの作成」→ エンジン選択 → テンプレート（本番/開発）→ マルチAZ・サブネットグループ・セキュリティグループを指定。
- パスワードは `--manage-master-user-password` で **Secrets Manager 管理**にしておくと、コードに平文を埋め込まずに済む。

## 実務での勘所
- **接続情報は環境変数 / Secrets Manager から**取得する。ソースに `password=...` を書かない。
- **リードレプリカは「参照専用」**。書き込みはプライマリのエンドポイントへ、参照系クエリはレプリカのエンドポイントへ振り分ける（アプリ側でDB接続を分ける）。
- **マルチAZは可用性、レプリカは負荷分散**。役割が違うので混同しない（マルチAZのスタンバイは普段クエリを受けない）。
- **Aurora を選ぶ場面**：スループットが高い／レプリカを素早く増やしたい／ストレージ自動拡張やフェイルオーバーの速さが欲しいとき。**Aurora Serverless v2** なら負荷に応じて容量が自動伸縮し、アクセスの波が大きいアプリに向く。
- 監視は CloudWatch（CPU・接続数・空きストレージ・レプリケーション遅延）。**接続数の枯渇**は事故の定番なので、アプリ側でコネクションプールを使う。→ [monitoring.md](./monitoring.md)

## ハマりどころ / アンチパターン
- **パブリックアクセス可で立ててしまう**：`--publicly-accessible` でグローバルに晒すと、総当たり攻撃の的。**DBはプライベートサブネットに置き、`--no-publicly-accessible` が原則**。接続はVPC内のアプリ or 踏み台/SSM経由で。→ [vpc.md](./vpc.md)
- **インスタンスサイズとコスト**：RDSは**起動しているだけで課金**（EC2と違い止めても7日で自動再起動）。開発用に大きいクラスを立てっぱなしにすると高い。小さく始めて必要時にスケールアップ。
- **Aurora と RDS の違いを誤解**：Auroraは**高機能だが最小構成でも割高**になりがち。小規模・低トラフィックなら通常のRDS（`db.t4g`系）の方が安いことが多い。要件で選ぶ。
- **メンテナンスウィンドウ無視**：RDSは指定した曜日・時間帯に**勝手にパッチ適用・再起動**することがある。ピーク時間を避けた**メンテナンスウィンドウ**を設定し、再起動が起き得る前提で接続リトライを実装する。
- **バックアップ保持0日**：`backup-retention-period 0` だと自動バックアップが無効で**PITRできない**。本番は最低7日程度を確保。
- **マルチAZにしていない本番**：単一AZだと障害＝即ダウン。本番は基本マルチAZ。

## 関連
[vpc.md](./vpc.md) / [dynamodb.md](./dynamodb.md) / [monitoring.md](./monitoring.md)
