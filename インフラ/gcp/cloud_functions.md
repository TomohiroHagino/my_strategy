# Cloud Functions（GCP）

## ひとことで言うと
**「関数1個」を書いてデプロイするだけで動く FaaS（Function as a Service）**。サーバもコンテナも自分で用意せず、**イベント（HTTPリクエストやファイル作成など）をトリガに、その関数だけが起動・実行**される。イベント駆動の小さな処理に向く。

## 役割・なぜ必要か
- やりたいことが「Webhookを受ける」「GCSにファイルが上がったらサムネ生成」「Pub/Subのメッセージを処理」程度なら、**アプリ全体を作るより関数1個で済む**。
- スケール・サーバ管理は不要。呼ばれた時だけ動き、暇なら0台＝課金もほぼゼロ（Cloud Runと同じ思想）。
- **「イベントに反応する糊（glue）コード」**を置く場所。サービス間をつなぐ小さな処理を、独立してデプロイ・課金できる。
- 2nd gen（第2世代）は内部的に **Cloud Run の上で動く**。つまり Cloud Run の自動スケール・同時実行・タイムアウトの仕組みをそのまま享受しつつ、「関数だけ書けばいい」手軽さを足したもの。

## 基本の使い方（gcloud / コンソール）
```bash
# HTTPトリガーの関数をデプロイ（2nd gen）
gcloud functions deploy my-webhook \
  --gen2 \
  --runtime nodejs20 \
  --region asia-northeast1 \
  --source . \
  --entry-point handler \       # 呼び出す関数名
  --trigger-http \
  --allow-unauthenticated

# イベントトリガー：GCSにファイルが作られたら起動
gcloud functions deploy on-upload \
  --gen2 --runtime nodejs20 --region asia-northeast1 \
  --source . --entry-point onUpload \
  --trigger-event-filters="type=google.cloud.storage.object.v1.finalized" \
  --trigger-event-filters="bucket=my-bucket"

# Pub/Sub のメッセージで起動
gcloud functions deploy on-message \
  --gen2 --runtime nodejs20 --region asia-northeast1 \
  --source . --entry-point onMessage \
  --trigger-topic my-topic
```

```javascript
// index.js : HTTP トリガーのエントリポイント
exports.handler = (req, res) => {
  const name = req.query.name || "world";
  res.status(200).send(`hello ${name}`);
};

// イベントトリガー（CloudEvent を受け取る）
exports.onUpload = (cloudEvent) => {
  const file = cloudEvent.data;     // バケット名・ファイル名など
  console.log(`uploaded: ${file.bucket}/${file.name}`);
};
```
- コンソールでは「Cloud Functions → 関数を作成」でランタイム・トリガー種別（HTTP / Pub/Sub / Storage 等）を選び、エディタで直接コードを書いてデプロイもできる。

## 実務での勘所
- **トリガーは大きく2種類**：①HTTPトリガー（URLが払い出され、リクエストで起動）②イベントトリガー（GCS・Pub/Sub・Firestore 等の出来事で起動）。用途で選ぶ。
- **ランタイム**：Node / Python / Go / Java など。エントリポイント（`--entry-point`）に呼び出す関数名を合わせる。
- **環境変数・シークレット**：DB接続情報などは `--set-env-vars` や Secret Manager 連携で渡す。コードに直書きしない。→ [iam.md](./iam.md)
- イベントトリガーは内部で **Eventarc / Pub/Sub** を経由する。GCSやFirestoreの変更を受けるには、対象サービス側のイベント発行設定や権限が要ることがある。

## ハマりどころ / アンチパターン
- **コールドスタート**：0台からの初回起動は遅い。重い依存ライブラリを減らす・関数を小さく保つ。レイテンシ必須なら最小インスタンスを設定（2nd genは設定可）。
- **タイムアウト**：関数には実行時間の上限がある。長時間処理を1関数で回そうとすると途中で切られる。重い処理は分割し、Pub/Subで非同期に連鎖させる。
- **Cloud Run との使い分けで迷う**：判断軸は「**関数1個で完結するイベント処理か / 複数エンドポイントを持つアプリ（コンテナ）か**」。前者は Functions、後者やDockerfileでコントロールしたいものはCloud Run。2nd genは実体がCloud Runなので、規模が育ったら Cloud Run へ移しやすい。→ [cloud_run.md](./cloud_run.md)
- **状態をローカルに持つ**：Cloud Run同様ステートレス前提。ローカルディスクや変数に状態を残さない（次回呼び出しで消える・別インスタンスに無い）。
- **冪等性（idempotency）を考えない**：イベントトリガーは**同じイベントが2回届く可能性**がある（at-least-once）。「2回実行されても壊れない」処理にしておく。
- **重い同期チェーンを関数で組む**：関数Aが関数Bを同期で呼んで…と連ねると、遅延とタイムアウトが累積する。間にPub/Subを挟んで疎結合に。

## 関連
[cloud_run.md](./cloud_run.md) / [iam.md](./iam.md) / [cloud_storage.md](./cloud_storage.md)
