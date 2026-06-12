# 実務でハマる罠まとめ（Pitfalls）（GCP）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。GCPは「コスト爆発」と「情報漏洩」の2大事故が多いので、構築前・公開前のセルフチェックに使う。

## 役割・なぜ必要か
- GCPは設定一つで高額課金や全世界公開になりうる。症状・場面から該当箇所へ素早く飛ぶための索引。
- 特に「課金」「公開範囲」「権限」は事故ると影響が大きいので最初に確認する。

## 権限 / IAM
- **基本ロール（Owner/Editor）の乱用**：広すぎる権限を人に付けると事故・漏洩の温床。役割ごとの事前定義ロール＋最小権限、人ではなくサービスアカウントに付与。→ [iam.md](./iam.md)
- **サービスアカウントキー（JSON）の漏洩**：鍵ファイルをコミット/共有すると即なりすまし。鍵は極力発行せず Workload Identity 等を使い、漏れたら即無効化・ローテーション。→ [iam.md](./iam.md)
- **サービスアカウントに過剰ロール**：Cloud Run / VM に紐づくSAが強権限だと、乗っ取り時の被害が拡大。用途別に最小権限のSAを分ける。→ [iam.md](./iam.md)

## 課金 / コスト
- **予算アラート未設定**：気づいたら高額請求。プロジェクト作成直後に予算とアラート（しきい値通知）を必ず設定。→ [getting_started.md](./getting_started.md)
- **VMの停止忘れ**：Compute Engine は起動中ずっと課金。検証用VMは使い終わったら停止/削除、自動停止スケジュールも検討。→ [getting_started.md](./getting_started.md) / [compute_engine.md](./compute_engine.md)
- **外部IP・ディスク・スナップショットの放置**：VMを消しても静的IPや永続ディスクが残り課金継続。リソース棚卸しを定期実施。→ [compute_engine.md](./compute_engine.md)
- **BigQuery `SELECT *` でフルスキャン**：オンデマンド課金はスキャン量課金。`SELECT *` や WHERE なしは高額。必要列だけ＋パーティション/クラスタで絞る。→ [bigquery.md](./bigquery.md)
- **ログのストレージ課金**：詳細ログを取りっぱなしで保持期間を放置すると蓄積課金。ログルーター/シンクと保持期間、不要ログの除外を設定。→ [monitoring.md](./monitoring.md)

## セキュリティ / 公開範囲
- **使うAPIの有効化し忘れ**：呼び出しが `API has not been used / disabled` で失敗。利用前に該当APIを有効化（`gcloud services enable`）。→ [getting_started.md](./getting_started.md)
- **Cloud Storage バケットの誤公開**：`allUsers`/`allAuthenticatedUsers` 付与や均一アクセス無効で全世界公開＝情報漏洩。公開防止設定・均一バケットレベルアクセスを徹底。→ [cloud_storage.md](./cloud_storage.md)
- **ファイアウォール `0.0.0.0/0` 全開放**：SSH(22)/RDP(3389)/DBポートを全開放すると総当たり・侵入の的。送信元IPを絞る or IAP/踏み台経由に。→ [vpc.md](./vpc.md) / [compute_engine.md](./compute_engine.md)
- **Cloud SQL をパブリックIP直結**：グローバルIPでDBを晒すのは危険。Cloud SQL Auth Proxy かプライベートIP（VPC経由）で接続する。→ [cloud_sql.md](./cloud_sql.md)

## サーバーレス / データストアの前提
- **Cloud Run はステートレス前提**：コンテナのローカルディスクは揮発し、インスタンス間で共有もされない。永続化は Cloud Storage / DB、状態は外部へ。→ [cloud_run.md](./cloud_run.md)
- **Cloud Run のコールドスタート/同時実行**：最小インスタンス0だと初回が遅い、`--concurrency` 設定で挙動が変わる。要件に応じて min-instances や同時実行数を調整。→ [cloud_run.md](./cloud_run.md)
- **Firestore はクエリの事前インデックス前提**：複合条件クエリは複合インデックスが必要で、無いと実行時エラー。事前に作成（エラーの作成リンク/`firestore.indexes.json`）。→ [firestore.md](./firestore.md)
- **Firestore は全文検索/集計が苦手**：`LIKE` 検索や横断集計は不得手。検索は外部（全文検索サービス）、件数は集計クエリ/カウンタで設計。→ [firestore.md](./firestore.md)

## IaC / 運用
- **IaC の drift（実態とコードの乖離）**：コンソールで手動変更するとTerraformの状態とズレ、次の `apply` で破壊/巻き戻し。変更はコード経由に統一、`plan` で差分確認。→ [iac.md](./iac.md)
- **tfstate の取り扱い**：state にシークレットが入りうる、ローカル保持は危険。リモートバックエンド（GCSバケット）＋ロックで管理。→ [iac.md](./iac.md)
- **リージョン/ゾーンの取り違え**：デフォルトリージョン未設定や不一致でリソースが想定外の場所に。`gcloud config` で既定を固定し明示指定。→ [getting_started.md](./getting_started.md)

## 関連
[iam.md](./iam.md) / [getting_started.md](./getting_started.md) / [compute_engine.md](./compute_engine.md) / [cloud_run.md](./cloud_run.md) / [cloud_storage.md](./cloud_storage.md) / [cloud_sql.md](./cloud_sql.md) / [firestore.md](./firestore.md) / [bigquery.md](./bigquery.md) / [vpc.md](./vpc.md) / [iac.md](./iac.md) / [monitoring.md](./monitoring.md)
