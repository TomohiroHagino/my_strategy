# Cloud Run（GCP）

## ひとことで言うと
**コンテナイメージを渡すだけで動く、サーバーレスのコンテナ実行基盤**。HTTPリクエストが来た瞬間にコンテナが起動し、負荷に応じて自動でスケール、暇になればゼロまで縮む（ゼロスケール）。GCPでアプリを動かすときの**主役**。

## 役割・なぜ必要か
- 「Dockerでコンテナにできるものは、ほぼそのまま動かせる」のが最大の利点。インフラ（VM・OS・スケーリング・ロードバランサ）を意識せず、**アプリのコンテナだけ**に集中できる。
- Compute Engine（VM）と違い、**起動・台数・冗長化を自前で管理しない**。トラフィックに応じて勝手に増減し、アクセスが無い時間は0台＝**課金もほぼゼロ**。
- 言語・フレームワーク不問。「`PORT` 環境変数で指定されたポートでHTTPを待ち受ける」という1点さえ守れば、Rails でも Go でも Node でも動く。
- リクエスト課金（使った分だけ）なので、**小〜中規模のWeb/API・社内ツール・バッチのHTTPトリガー**と相性が良い。

## 基本の使い方（gcloud / コンソール）
```dockerfile
# Dockerfile の要点：PORT を listen するだけ
FROM node:20-slim
WORKDIR /app
COPY . .
RUN npm ci --omit=dev
# Cloud Run は環境変数 PORT（既定8080）を渡してくる
ENV PORT=8080
CMD ["node", "server.js"]
```

```bash
# ① ソースから直接デプロイ（Dockerfile があれば自動ビルド）
gcloud run deploy my-api \
  --source . \
  --region asia-northeast1 \
  --allow-unauthenticated      # 未認証アクセスを許可（公開API）

# ② 既存イメージを指定してデプロイ
gcloud run deploy my-api \
  --image asia-northeast1-docker.pkg.dev/PROJECT/repo/my-api:latest \
  --region asia-northeast1 \
  --max-instances 10 \         # 上限を決めて暴走課金を防ぐ
  --concurrency 80 \           # 1インスタンスが同時にさばくリクエスト数
  --min-instances 1            # 常時1台（コールドスタート回避・有料）

# サービスのURL確認 / ログ確認
gcloud run services describe my-api --region asia-northeast1
gcloud run services logs read my-api --region asia-northeast1
```
- コンソールでは「Cloud Run → サービスを作成」でイメージ選択・リージョン・認証・スケール設定をGUIで指定できる。

## 実務での勘所
- **リビジョン**：デプロイのたびに不変の「リビジョン」が作られる。設定とイメージのスナップショットなので、**問題があれば前リビジョンに即ロールバック**できる。
- **トラフィック分割**：複数リビジョンに割合を振れる（カナリアリリース）。例：新リビジョンに10%だけ流して様子を見る。
  ```bash
  gcloud run services update-traffic my-api \
    --to-revisions my-api-00007-abc=10,my-api-00006-xyz=90 \
    --region asia-northeast1
  ```
- **最小インスタンス（`--min-instances`）**：1以上にすると常駐し、コールドスタートが消える代わりに**待機中も課金**。レイテンシ重視のAPIで使う。
- **同時実行数（`--concurrency`）**：1インスタンスが同時に処理するリクエスト数。CPU/メモリを食う処理は小さく、軽いI/O中心なら大きく。
- 認証は **IAM** で制御。公開しないなら `--no-allow-unauthenticated` にして、呼び出し側にロール（`roles/run.invoker`）を付与する。→ [iam.md](./iam.md)

## ハマりどころ / アンチパターン
- **コールドスタート**：アクセスが無いと0台→次の1発目はコンテナ起動分だけ遅い。レイテンシが効くなら `--min-instances 1` で常駐させる。イメージを軽くするのも効く。
- **ステートレス前提を破る**：コンテナの**ローカルディスクは揮発**（書いても次のリクエストや別インスタンスには残らない）。アップロードファイルは [Cloud Storage](./cloud_storage.md)、セッションやキャッシュは外部（DB/Redis）へ。「インスタンスはいつ消えてもいい」を前提に設計する。
- **インスタンスをまたぐ状態を前提にする**：自動スケールで台数は常に変わる。**メモリ上のグローバル変数で状態を共有しない**（インスタンスごとに別物）。
- **リクエストタイムアウト**：既定は短め（最大でも上限あり）。長時間バッチをHTTPで処理しようとすると途中で切られる。重い処理は分割するか、Pub/Sub＋非同期、ジョブ系へ。
- **`PORT` を無視**：ハードコードしたポートで待つと起動チェックに失敗する。必ず `process.env.PORT`（既定8080）を listen する。
- **`--max-instances` 無制限のまま**：急なアクセス急増でインスタンスが増えすぎ、下流のDBコネクションを枯渇させたり課金が跳ねる。上限を必ず決める。

## 関連
[cloud_functions.md](./cloud_functions.md) / [iam.md](./iam.md) / [cloud_storage.md](./cloud_storage.md)
