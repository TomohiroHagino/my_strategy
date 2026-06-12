# Active Storage（Rails 6）

## ひとことで言うと
Rails 標準のファイル添付機能。モデルに `has_one_attached` / `has_many_attached` を宣言し、ローカルや S3 などのストレージへファイルを保存・配信する。画像は variant でリサイズできる。

## 役割・なぜ必要か
- プロフィール画像や添付ファイルの「アップロード→保存→表示」を標準機能で扱える。
- 保存先（service）を local / amazon(S3) / google / microsoft から `config/storage.yml` で切り替えられる。コードは変えずに本番だけ S3 にできる。
- 大きなファイルはサーバを経由せずクライアントから直接アップロードする direct upload に対応。

## 基本の書き方（コード）
```bash
bin/rails active_storage:install
bin/rails db:migrate
```
```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_one_attached :avatar
  has_many_attached :images
end
```
```erb
<%# フォーム %>
<%= form_with model: @user do |f| %>
  <%= f.file_field :avatar %>
  <%= f.file_field :images, multiple: true %>
<% end %>
```
```ruby
# 添付・表示
user.avatar.attach(params[:user][:avatar])
# Strong Parameters: params.require(:user).permit(:avatar, images: [])
```
```erb
<%= image_tag user.avatar if user.avatar.attached? %>
<%# variant（image_processing gem が必要） %>
<%= image_tag user.avatar.variant(resize_to_limit: [100, 100]) %>
```
```yaml
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>
amazon:
  service: S3
  bucket: my-bucket
  region: ap-northeast-1
```
```ruby
# config/environments/production.rb
config.active_storage.service = :amazon
```
```erb
<%# direct upload（Webpacker の activestorage JS が必要） %>
<%= f.file_field :avatar, direct_upload: true %>
```

## 実務での使い方・定番パターン
- プロフィール画像（`has_one_attached`）、複数添付ファイル（`has_many_attached`）。
- 本番は S3、開発は local に分ける。`image_processing`（MiniMagick / libvips）でサムネイル生成。
- 一覧表示の N+1 は `User.with_attached_images` でまとめて読み込む。

## ハマりどころ / アンチパターン
- `image_processing` gem 未導入だと `variant` が失敗する。Gemfile に追加し ImageMagick / libvips をインストールする。
- 本番で `config.active_storage.service` を S3 に設定し忘れると、ローカルディスクに保存され再デプロイで消える。
- direct upload は別オリジンへ送るため CORS 設定（S3 バケット側）が必要。
- ファイルサイズ・拡張子・MIME のバリデーションは標準で行われない。モデルで自前検証（または `active_storage_validations` gem）する。

## 関連
[action_text.md](./action_text.md) / [webpacker.md](./webpacker.md) / [model.md](./model.md) / [view.md](./view.md) / [strong_parameters.md](./strong_parameters.md) / [config_credentials.md](./config_credentials.md) / [security.md](./security.md) / [pitfalls.md](./pitfalls.md)
