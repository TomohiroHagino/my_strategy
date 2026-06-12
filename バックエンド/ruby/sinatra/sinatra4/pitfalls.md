# 実務でハマる罠まとめ（Pitfalls）＋ Rails との使い分け（Sinatra 3/4）

## ひとことで言うと
Sinatraの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。
Sinatraは「規約が無い＝自由」ゆえに、**自分で守らないと事故る**箇所が多い。症状から該当箇所へ素早く飛ぶための索引。

## 役割・なぜ必要か
- Railsと違い枠組みが薄いので、ルーティング順・DB・XSS・構造などを**自前で意識**する必要がある。
- 「動くけど危ない / 後で詰む」書き方を、レビュー前のセルフチェックで弾く。

## ルーティング
- **定義順で最初にマッチが勝つ**：上から評価され、最初に一致したルートで止まる。`/posts/:id` を `/posts/new` より上に書くと `new` が `:id="new"` に食われる。**具体的なルートを先、可変を後**に。→ [routing.md](./routing.md)
- **ワイルドカード `*` の取りこぼし**：`/files/*` は `params[:splat]`（配列）で受ける。`params[:captures]` との違いに注意。→ [routing.md](./routing.md)
- **末尾スラッシュ**：`/foo` と `/foo/` は別ルート。両対応するなら `/foo/?` のように書く。→ [routing.md](./routing.md)

## データアクセス
- **ORMは内蔵されていない＝自前**：DBが要るなら ActiveRecord か Sequel を**自分で gem 追加・接続設定**する。Railsの感覚で「いきなりモデルが使える」と思うと詰む。→ [data_access.md](./data_access.md)
- **接続プール / 切断**：Webサーバ（Puma等）のスレッド/プロセスとDB接続数を自前で整合させる。マイグレーションも自前タスク。→ [data_access.md](./data_access.md)

## ビュー / セキュリティ
- **ERBは自動エスケープしない＝XSS**：RailsのERBと違い、Sinatra標準のERBは `<%= user_input %>` を**そのまま出力**する。`<%==` で生HTML、`<%= %>` でもエスケープされない点に注意。`Rack::Utils.escape_html` や `erb`+`escape_html: true` 設定、もしくは ERB の `<%= h(...) %>` 相当を用意する。→ [views.md](./views.md)
- **テンプレートにユーザ入力を直接埋め込まない**：パス・ファイル名を `params` から組むとテンプレートインジェクション/パストラバーサルの恐れ。→ [views.md](./views.md)

## セッション / CSRF
- **セッションは明示有効化**：`enable :sessions` しないと `session` は使えない。さらに `session_secret` を本番で明示しないと再起動で失効。→ [config_testing.md](./config_testing.md)
- **CSRF保護は標準で入らない**：状態変更フォームには `Rack::Protection`（`rack-protection` gem）を**自分で組み込む**。Railsのような自動CSRFトークンは無い。→ [rack_and_filters.md](./rack_and_filters.md)

## 構造 / スタイル
- **classic と modular の混同**：1ファイルの classic（トップレベルに `get`）と、`Sinatra::Base` 継承の modular を混ぜると `settings` / 起動方法が噛み合わず動かない。プロジェクト方針を最初に固定。→ [getting_started.md](./getting_started.md)
- **大きくなると自前構造の負担**：ルートが増えると1ファイルが肥大化。helper・ルート分割・サービス層などを**規約なしで自分で設計**する必要があり、統制が崩れやすい。早めに modular + ファイル分割へ。
- **本番で例外が隠れる**：`production?` では既定で `show_exceptions` が無効。ログ（`settings.logging`）とエラーハンドラ（`error do ... end`）を自前で用意しないと原因が掴めない。→ [config_testing.md](./config_testing.md)

## Sinatra vs Rails 使い分け
| 状況 | 選ぶもの | 理由 |
|------|----------|------|
| 小さな単機能・Webhook受け口・社内小物 | **Sinatra** | 起動が軽く、URL↔処理を数行で書ける。魔法が少なく透明。 |
| 単純なAPI（数エンドポイント） | **Sinatra** | ルーティングとJSON返却だけで足り、全部入りは過剰。 |
| プロトタイプ / 学習 | **Sinatra** | 構成が読みやすく、捨てやすい。 |
| CRUDが多い・管理画面が要る | **Rails** | scaffold・Active Record・admin系gemで規約に沿って速く作れる。 |
| 認証・メール・ジョブ・国際化など定番機能が多い | **Rails** | 周辺が標準/準標準で揃い、自前実装の手間が消える。 |
| チーム開発で統一規約が要る | **Rails** | 「規約が構造を強制」するため属人化しにくい。 |
| Sinatraが肥大化してきた | **Railsへ移行を検討** | ルート/モデル/構造を自前で支える負担が、Railsの規約コストを上回ったら乗り換え時。 |

**目安**：ファイル数・エンドポイント数・「自前で作った仕組み（認証・ORM接続・構造）」が増え、規約が欲しくなったらRailsへ。逆に「Railsは重い・1機能だけ」ならSinatra。

## 関連
[getting_started.md](./getting_started.md) / [routing.md](./routing.md) / [views.md](./views.md) / [data_access.md](./data_access.md) / [rack_and_filters.md](./rack_and_filters.md) / [config_testing.md](./config_testing.md)
> 本格版Railsの罠は [../rails/rails7/pitfalls.md](../../rails/rails7/pitfalls.md) を参照。
