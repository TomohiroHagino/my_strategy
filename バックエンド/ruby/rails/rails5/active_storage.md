# Active Storage（Rails 5.2）

## ひとことで言うと
**ファイルアップロードを Rails 標準で扱う仕組み**。画像やPDFなどをモデルに紐づけて、ローカルディスク / Amazon S3 / Google Cloud Storage 等へ保存できる。**Rails 5.2 で追加**された（5.0/5.1 には無い）。

## 役割・なぜ必要か
- それまでファイルアップロードは CarrierWave / Paperclip などの外部gemに頼っていた。Active Storage はこれを Rails 標準として取り込み、保存先（ローカル/クラウド）を設定で切り替えられるようにした。
- 添付ファイルのメタデータをDBで管理し、`user.avatar` のようにモデル経由で扱える。

## セットアップ
```bash
rails active_storage:install   # 5.2。active_storage_blobs / attachments テーブルを作る
rails db:migrate
```
```yaml
# config/storage.yml（保存先の定義）
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>
amazon:
  service: S3
  bucket: my-bucket
  region: ap-northeast-1
  # access_key_id / secret_access_key は credentials から読む
```
```ruby
# config/environments/production.rb
config.active_storage.service = :amazon   # 本番は S3 など
```

## 基本の書き方（コード）
```ruby
# モデル
class User < ApplicationRecord
  has_one_attached  :avatar       # 1ファイル
  has_many_attached :documents    # 複数ファイル
end
```
```erb
<%# フォーム %>
<%= form_with model: @user, local: true do |f| %>
  <%= f.file_field :avatar %>
  <%= f.submit %>
<% end %>
```
```ruby
# コントローラ（Strong Parameters で許可）
params.require(:user).permit(:name, :avatar)
```
```erb
<%# 表示（バリアント＝リサイズには image_processing gem が必要） %>
<%= image_tag @user.avatar if @user.avatar.attached? %>
<%= image_tag @user.avatar.variant(resize: "100x100") %>
```

## 実務での使い方・定番パターン
- **本番は S3 等のクラウド**、開発は `Disk`。`config.active_storage.service` を環境ごとに切り替える。
- **認証情報は credentials**（`aws: access_key_id: ...`）に置き、`storage.yml` から参照。→ [config_credentials.md](./config_credentials.md)
- **画像のリサイズ**は `variant` ＋ `image_processing` gem（MiniMagick / Vips）。
- **直接アップロード**（ブラウザ→ストレージへ直送）で大きいファイルのサーバ負荷を避けられる（`direct_upload: true`）。
- **添付の有無チェック**：表示前に `attached?` で確認しないと nil で落ちる。

## ハマりどころ / アンチパターン
- **5.0/5.1 には無い**：Active Storage は 5.2 から。それ以前は CarrierWave / Paperclip を使う（バージョン取り違え注意）。
- **`install` 忘れ**：テーブルが無いと添付保存で落ちる。`active_storage:install` ＋ migrate を実行。
- **`variant` に gem が要る**：`image_processing`（と MiniMagick / libvips）が無いとリサイズで例外。
- **ファイル種別/サイズの検証が標準では弱い**：5.2 時点では組み込みバリデーションが薄いので、独自に拡張子・サイズ・MIMEを検証する。→ [security.md](./security.md)
- **N+1**：一覧でサムネイル表示すると添付取得でクエリが増える。`with_attached_avatar` 等で eager load する。

## 関連
[model.md](./model.md) / [config_credentials.md](./config_credentials.md) / [security.md](./security.md)
