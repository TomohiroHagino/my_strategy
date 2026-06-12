# インフラ（クラウド）

アプリを動かす土台＝クラウド基盤のリファレンス。AWS と GCP を、アプリ開発者が**実際に使うコアサービス**中心に項目別でまとめる。

## クラウドの基本概念（AWS/GCP共通）
- **リージョン / ゾーン**：データセンターの地理的なまとまり（東京等）。近い所を選ぶ。
- **IAM（認証・認可）**：誰が何をできるか。**最小権限**が鉄則。
- **コンピュート**：VM / コンテナ / サーバーレス（FaaS）の3系統。
- **ストレージ**：オブジェクト（S3/GCS）/ ブロック / DB（RDB・NoSQL）。
- **ネットワーク**：VPC（仮想ネットワーク）/ CDN / DNS。
- **IaC（Infrastructure as Code）**：構成をコードで管理（Terraform 等）。
- **課金**：従量課金。**コスト監視とアラートが必須**（事故りやすい）。

## AWS と GCP（ざっくり対応）
| 用途 | AWS | GCP |
|---|---|---|
| VM | EC2 | Compute Engine |
| サーバーレス関数 | Lambda | Cloud Functions |
| コンテナ実行 | ECS/Fargate | **Cloud Run** |
| オブジェクト保存 | S3 | Cloud Storage |
| RDB | RDS/Aurora | Cloud SQL |
| NoSQL | DynamoDB | Firestore |
| データ分析 | Athena/Redshift | **BigQuery** |
| IAM/DNS/CDN | IAM / Route53 / CloudFront | IAM / Cloud DNS / Cloud CDN |

## このフォルダの構成
- [aws/](./aws/) … **Amazon Web Services** リファレンス（IAM / EC2 / Lambda / S3 / RDS / VPC …）
- [gcp/](./gcp/) … **Google Cloud Platform** リファレンス（IAM / Compute / Cloud Run / GCS / BigQuery …）
- [terraform/](./terraform/) … **Terraform**（クラウド横断のIaC）リファレンス。宣言的に「あるべき状態」を `.tf` で書き、`init → plan → apply` で適用する。AWS/GCPどちらも同じ書き方で扱える（Provider / State / Module / 環境分け …）
- [webサーバ/](./webサーバ/) … **Webサーバ / リバースプロキシ** リファレンス。アプリの前段に立つ受付係＝**nginx / Apache**。静的配信・**TLS終端**・複数アプリへの**振り分け（リバースプロキシ）**・**ロードバランス**を担う。`ブラウザ → nginx/Apache → アプリサーバ(Puma/Gunicorn/Node…)` の構図（nginx / apache / リバースプロキシ）。クラウドではALB/CloudFrontがこの役割を吸収することも。
- [docker/](./docker/) … **Docker（コンテナ）** リファレンス。アプリを実行環境ごと箱（イメージ）に詰めて「自分の環境では動く」を消す再現性の道具。`Dockerfile → build → image → run → container` の流れを軸に、Dockerfile / Compose / Volume・Network / コマンド / ベストプラクティス / 本番運用 …。本番のオーケストレーション（k8s/ECS）は [aws/containers.md](./aws/containers.md) へ。
- [障害対応/](./障害対応/) … **運用・インシデント対応（SRE）** リファレンス。本番障害を「検知 → 初動 → 切り分け → 復旧 → 再発防止」の一連で回す型。重大度(SEV)判定・止血優先・ブレームレスなポストモーテム・ランブック（初動対応 / 切り分け / 復旧手順 / よくある障害パターン …）
- [負荷検証/](./負荷検証/) … **性能・キャパシティ（負荷試験・パフォーマンステスト）** リファレンス。本番相当(staging)にわざと負荷をかけ「壊れる前に限界と性能を知る」。「計画 → 実施 → 分析 → 改善」で回す。負荷の種類(ロード/ストレス/スパイク/ソーク/キャパシティ)・指標(RPS・**レイテンシ p95/p99**・エラー率・**飽和点**・Little's Law)・**k6** など（種類と目的 / 指標 / 計画 / ツール / 実施手順 / 結果分析 / チューニング …）。分析・改善は [障害対応/切り分け.md](./障害対応/切り分け.md) と接続。

> ※ クラウドはサービスが継続的に増減・変更される。本書は概念とコアサービスの理解が主目的。料金や最新仕様は公式で確認。
