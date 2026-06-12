# Ruby on Rails

## 一言で
Rubyの代表的なフルスタックWebフレームワーク。「**設定より規約（CoC）**」「**DRY**」を徹底し、少ないコードでDB駆動のWebアプリを高速に構築できる。

## 特徴
- **MVC アーキテクチャ**: Model / View / Controller の明確な分担。
- **Active Record（ORM）**: テーブル＝クラス、行＝オブジェクト。SQLをほぼ書かずにDB操作。
- **Convention over Configuration**: 命名規約に従えば設定ほぼ不要（`User`モデル→`users`テーブル等）。
- **オールインワン**: ルーティング・ORM・ビュー・メール・ジョブ（Active Job）・テストまで同梱。
- **generator / scaffold / migration**: 雛形生成とDBスキーマのバージョン管理が標準。
- **Hotwire（Turbo / Stimulus）**: JSを最小限にしつつSPA的なUI/部分更新を実現。

## どういう使い方をするのか
- **DB駆動のWebアプリ全般**: SaaS、管理画面、EC、業務システム、社内ツール。
- **MVP / スタートアップの主力**: モノリスで素早くプロダクトを立ち上げる。
- **API モード**: ビューを省き、フロント（React/Next等）やモバイル向けのバックエンドAPIとして。

## 強み
- 立ち上げ速度が圧倒的。規約に乗れば「決め事」で迷わない。
- Active Record が強力で、CRUD・リレーションが書きやすい。
- セキュリティ（CSRF/SQLi/XSS対策）やテストのデフォルトが堅実。
- Hotwireでフロントの負担を抑えたまま今風のUXを出せる。
- 成熟したgemエコシステムで「やりたいこと」に大抵gemがある。

## 弱み・注意点
- **規約から外れると一気に辛くなる**（レールを降りるコスト）。
- Active Record の **N+1 / 重さ**、暗黙の「マジック」挙動の学習コスト。
- モノリスが肥大化しやすい。大規模・高トラフィックではスケール設計の工夫が要る。

## 向いている場面 / 向かない場面
- 向いている: スタートアップMVP、CRUD中心のSaaS・業務アプリ、少人数で速く作る案件。
- 向かない: 極限の性能要求、超大規模分散、大量同時接続のリアルタイム（別途設計が必要）。

## エコシステム・周辺ツール
- 認証: Devise / 認可: Pundit・CanCanCan
- 非同期ジョブ: Sidekiq（+ Redis）
- テスト: RSpec / FactoryBot / Capybara
- フロント: Hotwire（Turbo / Stimulus）、ファイル: Active Storage
- デプロイ: Kamal / Capistrano

## ひとことメモ（自分の実感）
- （現役として触った所感を後から追記）

## このフォルダの構成（各版とも独立した実務リファレンス）
- [rails7/](./rails7/) … **Rails 7**（Hotwire / importmap / Zeitwerk 時代）
- [rails6/](./rails6/) … **Rails 6**（Webpacker標準 / Zeitwerk / 複数DB / Action Text・Mailbox）
- [rails5/](./rails5/) … **Rails 5**（APIモード / Action Cable / rails-ujs / credentials 5.2）
- [rails4/](./rails4/) … **Rails 4**（Sprocketsのみ / jquery_ujs / Turbolinks Classic / secrets.yml）
- [周辺インフラ/](./周辺インフラ/) … 版共通の周辺ミドルウェア（Redis / Sidekiq / Solid Queue / Unicorn …）

> 各版は「プロジェクトの始め方〜MVC〜テスト〜罠まで、項目=1ファイル」で同じ書式。
> バージョン差（アセット・JS・autoload・認証情報など）はその版の仕様に沿って記述。
