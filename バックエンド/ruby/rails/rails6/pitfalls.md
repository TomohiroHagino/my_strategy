# 実務でハマる罠まとめ（Rails 6 早見表）

## ひとことで言うと
Rails 6 特有・実務頻出の罠を分野別に並べた早見表。各行は「症状／原因／対処／詳細リンク」で、原因の切り分けと一次対処を素早く行うためのもの。

## Zeitwerk / オートロード

| 症状 | 原因 | 対処 | 詳細 |
|------|------|------|------|
| `NameError: expected file .../foo.rb to define constant Foo` | ファイル名と定数名の不一致 | パスから期待される定数名にクラス/モジュール名を合わせる | [zeitwerk.md](./zeitwerk.md) |
| `expected to define constant ApiController` だが `APIController` と書いた | 頭字語の inflection 未設定 | `inflections.rb` に `inflect.acronym "API"` を追加 | [zeitwerk.md](./zeitwerk.md) |
| 編集が反映されない / `superclass mismatch` | `app/` 配下を `require` している | `require` を削除し定数参照のみにする | [zeitwerk.md](./zeitwerk.md) |
| 本番起動で即クラッシュ | eager_load で命名ズレが一括検出 | デプロイ前に `bin/rails zeitwerk:check` | [zeitwerk.md](./zeitwerk.md) |

## Webpacker / JavaScript

| 症状 | 原因 | 対処 | 詳細 |
|------|------|------|------|
| `Webpacker::Manifest::MissingEntryError` | yarn 未インストール / precompile 未実行 | `yarn install` 後 `bin/rails webpacker:compile` | [webpacker.md](./webpacker.md) |
| JS が読み込まれない | `javascript_include_tag` を使い `javascript_pack_tag` を使っていない | pack は `javascript_pack_tag "application"` で読む | [webpacker.md](./webpacker.md) / [javascript.md](./javascript.md) |
| 本番 `assets:precompile` がメモリ不足で失敗 | webpack のメモリ消費大 | `NODE_OPTIONS=--max-old-space-size` 調整 / ビルド分離 | [webpacker.md](./webpacker.md) |
| ページ遷移後に JS 初期化が走らない | Turbolinks がフルリロードしないため `DOMContentLoaded` が発火しない | `document.addEventListener("turbolinks:load", ...)` で初期化 | [javascript.md](./javascript.md) |
| 開発で JS 変更が遅い | dev-server 未起動 | `bin/webpack-dev-server` を起動 | [webpacker.md](./webpacker.md) |

## 複数 DB（Multiple Databases）

| 症状 | 原因 | 対処 | 詳細 |
|------|------|------|------|
| 書込直後に読めない | replica（reader）のレプリケーション遅延 | 直後の読取は primary に向ける / 自動切替の遅延閾値を調整 | [multiple_databases.md](./multiple_databases.md) |
| 一部テーブルが migrate されない | migrate 対象 DB の指定漏れ | `bin/rails db:migrate:primary` 等で対象 DB を明示 | [multiple_databases.md](./multiple_databases.md) |
| 接続先が分かれない | 抽象基底クラスの `connects_to` 継承を使っていない | DB ごとの抽象クラスを作りモデルを継承させる | [multiple_databases.md](./multiple_databases.md) |
| 意図せず reader に流れる/流れない | 自動切替（`database_selector`）の条件 | 切替ミドルウェアの設定（GET は reader 等）を確認 | [multiple_databases.md](./multiple_databases.md) |

## ActiveRecord / N+1

| 症状 | 原因 | 対処 | 詳細 |
|------|------|------|------|
| クエリが件数分発行され遅い | 関連の遅延ロード（N+1） | `includes(:assoc)` でプリロード | [active_record.md](./active_record.md) |
| 6.1：意図しない関連ロードを検出したい | N+1 をテストで落としたい | `strict_loading` を有効化し未プリロード時に例外化 | [active_record.md](./active_record.md) |

## View / フォーム

| 症状 | 原因 | 対処 | 詳細 |
|------|------|------|------|
| `form_with` が ajax 送信になり画面が更新されない | 6.0 では `form_with` が既定 `remote: true` | 通常送信にするなら `local: true` を付ける | [view.md](./view.md) |
| 削除リンクが GET になり削除されない | `method: :delete` 未指定 / rails-ujs 未読込 | `link_to "削除", path, method: :delete` とし rails-ujs を読み込む（turbo_method ではない） | [javascript.md](./javascript.md) / [view.md](./view.md) |

## Strong Parameters

| 症状 | 原因 | 対処 | 詳細 |
|------|------|------|------|
| 値が保存されない（nil のまま） | `permit` にカラム追加漏れ | 保存対象カラムを `permit` に列挙する | [strong_parameters.md](./strong_parameters.md) |

## Controller / フィルタ

| 症状 | 原因 | 対処 | 詳細 |
|------|------|------|------|
| `DoubleRenderError`（二重 render/redirect） | `render`/`redirect_to` 後に `return` を忘れ処理が続行 | 分岐後に `return` を付ける | [controller.md](./controller.md) / [filters.md](./filters.md) |

## 認証情報 / credentials

| 症状 | 原因 | 対処 | 詳細 |
|------|------|------|------|
| 起動時にクラッシュ（credentials 復号不可） | `config/master.key` / `RAILS_MASTER_KEY` 不在 | 鍵を配置（コミットせず安全に共有） | [config_credentials.md](./config_credentials.md) / [security.md](./security.md) |

## ジョブ / メール

| 症状 | 原因 | 対処 | 詳細 |
|------|------|------|------|
| `deliver_later` が即時（同期）送信になる | 本番のキューアダプタ未設定（`:async` のまま等） | Sidekiq/Resque 等を `config.active_job.queue_adapter` に設定 | [active_job.md](./active_job.md) / [action_mailer.md](./action_mailer.md) |

## 関連
[zeitwerk.md](./zeitwerk.md) / [webpacker.md](./webpacker.md) / [javascript.md](./javascript.md) / [multiple_databases.md](./multiple_databases.md) / [active_record.md](./active_record.md) / [view.md](./view.md) / [controller.md](./controller.md) / [filters.md](./filters.md) / [strong_parameters.md](./strong_parameters.md) / [config_credentials.md](./config_credentials.md) / [active_job.md](./active_job.md) / [action_mailer.md](./action_mailer.md) / [security.md](./security.md) / [testing.md](./testing.md)
