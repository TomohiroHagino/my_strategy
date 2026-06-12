# Sinatra（Ruby のマイクロWebフレームワーク）

## 一言で
**最小限の“マイクロ”Webフレームワーク**。「URL ↔ 処理」を数行で書けるのが身上。Railsのような全部入り（ORM・admin・generator）は**持たず**、必要な物だけ自分で足す。小さなAPI・1ファイルアプリ・プロトタイプに最適。

## 特徴
- **超軽量・薄い**：`get "/" do ... end` だけでルートが書ける。
- **Rack の上の薄い層**：ミドルウェアやサーバ（Puma等）はRackの作法そのまま。
- **ORM等は内蔵しない**：DBが要るなら ActiveRecord / Sequel を**自分で組み合わせる**。
- **2スタイル**：1ファイルの **classic** と、クラスにする **modular**（`Sinatra::Base`）。

## どういう使い方をするのか
- 小さな**REST API**・Webhook受け口・管理用の小物。
- Railsを入れるほどでない**単機能アプリ / プロトタイプ**。
- マイクロサービスの1個。

## 強み / 弱み
- 強み：学習が速い・起動が軽い・構成が透明（魔法が少ない）。
- 弱み：大きくなると**自前で構造を作る負担**が増える（規約が無い）。本格的になったらRailsを検討。

## このフォルダの構成（フラッグシップ＝項目別・身の丈サイズ）
> Sinatraは小さいので版フォルダは作らず、ここに直接ファイルを置く（Sinatra 3/4 系想定）。

- [getting_started.md](./getting_started.md) … 始め方（gem / classic・modular / 起動）
- [request_flow.md](./request_flow.md) … リクエスト/処理の流れ・各層は何を返すか（全体像）
- [routing.md](./routing.md) … ルーティングとは（get/post・パラメータ・ワイルドカード）
- [request_response.md](./request_response.md) … リクエスト/レスポンス（params・halt・redirect・status）
- [views.md](./views.md) … ビュー（ERB・レイアウト・インラインテンプレート）
- [data_access.md](./data_access.md) … DB/ORM（内蔵なし＝ActiveRecord/Sequelを足す）
- [rack_and_filters.md](./rack_and_filters.md) … Rack・ミドルウェア・before/after フィルタ
- [config_testing.md](./config_testing.md) … 設定（settings/environments）とテスト（Rack::Test）
- [rack_test.md](./rack_test.md) … Rack::Test（get/post ヘルパ・last_response でアプリを直接叩く）
- [pitfalls.md](./pitfalls.md) … ハマり所＋Rails との使い分け

> 関連: 同じRubyの本格版は [../rails/](../../rails/)。周辺インフラ（Redis等）は [../rails/周辺インフラ/](../../rails/周辺インフラ/) を流用。
