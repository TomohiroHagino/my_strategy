# Cloud Storage（GCP）

## ひとことで言うと
**ファイル（オブジェクト）を無制限に置ける、クラウドのオブジェクトストレージ**。「**バケット**」という入れ物の中に「**オブジェクト**（ファイル）」を入れる。画像・動画・バックアップ・ログ・静的サイトの配信など、とにかく**モノを置く**ための場所。

## 役割・なぜ必要か
- Cloud Run / Cloud Functions は**ローカルディスクが揮発**するため、永続化したいファイルは外に置く必要がある。その置き場の定番がCloud Storage。→ [cloud_run.md](./cloud_run.md)
- 「フォルダ階層を持つファイルシステム」ではなく、**キー（オブジェクト名）と中身のペア**を保存する仕組み。`images/2026/a.png` のような名前で擬似的に階層風に見せるだけ。
- HTTP(S)で直接配信でき、画像や静的ファイルの**CDN的な配信元**になる。耐久性が非常に高く、容量を気にせず預けられる。
- アクセス頻度に応じて**ストレージクラス**を選び、コストを最適化できる（よく使うものは速く高く、滅多に使わないものは安く）。

## 基本の使い方（gcloud / コンソール）
```bash
# バケット作成（名前はグローバル一意・ロケーションを指定）
gcloud storage buckets create gs://my-app-assets \
  --location asia-northeast1 \
  --uniform-bucket-level-access      # 公開事故を防ぐ推奨設定（後述）

# アップロード / ダウンロード / 一覧 / 削除
gcloud storage cp ./photo.png gs://my-app-assets/images/photo.png
gcloud storage cp gs://my-app-assets/images/photo.png ./photo.png
gcloud storage ls gs://my-app-assets/images/
gcloud storage rm gs://my-app-assets/images/photo.png

# ストレージクラスを指定して作る（アクセス頻度で選ぶ）
gcloud storage buckets create gs://my-archive \
  --location asia-northeast1 \
  --default-storage-class COLDLINE

# 署名付きURL：限られた時間だけ非公開オブジェクトにアクセスさせる
gcloud storage sign-url gs://my-app-assets/images/photo.png \
  --duration 15m
```
- コンソールでは「Cloud Storage → バケット作成」でロケーション・ストレージクラス・アクセス制御をGUIで設定し、ドラッグ&ドロップでアップロードできる。

## 実務での勘所
- **ストレージクラス**：アクセス頻度で使い分けてコストを下げる。
  - `Standard`：頻繁に読む（Webの画像、配信ファイル）。
  - `Nearline`：月1回程度（少し古いログ、バックアップ）。
  - `Coldline`：四半期に1回程度（長期保管）。
  - （さらに低頻度の `Archive` もある）。取り出すほど安いクラスは**読み出しコストが高い**点に注意。
- **静的配信**：画像や静的サイトの配信元にできる。本番ではCDN（Cloud CDN）やロードバランサ経由にすると速く・安くなる。
- **署名付きURL（Signed URL）**：バケットを公開せずに、**「この1ファイルに、15分だけアクセスできるURL」**を発行する仕組み。ユーザーのアップロード受付やプライベートな配信に使う定番。公開設定をいじらずに済むので安全。
- **ライフサイクル管理**：「90日経ったらColdlineへ」「365日で削除」などのルールを設定して、放置データのコストを自動で抑える。

## ハマりどころ / アンチパターン
- **バケット誤公開で情報漏洩**：最大の事故。`allUsers` に読み取り権限を付けると**インターネット全公開**になる。個人情報や内部ファイルが丸見えになるニュースは大体これ。**`uniform bucket-level access`（均一なバケットレベルアクセス）を有効化**し、IAMで一元管理して、安易に公開しない。公開が必要な配信は署名付きURLやCDN経由を優先。→ [iam.md](./iam.md)
- **バケット名はグローバル一意**：プロジェクト内ではなく**全世界で重複不可**。`test` のような名前は取れない。`会社名-用途` のように衝突しない命名にする。
- **ロケーション選択を間違える**：作成後にロケーションは変更できない。アプリ（Cloud Run等）と**同じリージョン**に置かないと、通信が遅くなり**リージョン跨ぎの転送料**もかかる。
- **オブジェクトに「フォルダ」があると誤解**：名前の `/` は見た目だけ。大量オブジェクトのリネーム＝全コピー＆削除になり遅い。設計時に名前規約を決めておく。
- **公開URLで機密を配る**：「URLを知っている人だけ」は秘密ではない。共有・ログ・キャッシュから漏れる。時間制限のある署名付きURLか、IAM認証付き配信にする。

## 関連
[cloud_sql.md](./cloud_sql.md) / [cloud_run.md](./cloud_run.md) / [iam.md](./iam.md)
