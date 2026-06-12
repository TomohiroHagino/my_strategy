# Laravel

## 一言で
PHPで最も人気のフルスタックWebフレームワーク。「**開発者体験（DX）**」を重視し、表現力の高い文法と充実した標準機能（ORM・テンプレート・認証・キュー・CLI）で、Webアプリを高速に作れる。Railsから多くの思想を受け継ぐ。

## 特徴
- **Eloquent ORM**: Active Recordパターンのモデル。`User::where(...)->get()` のように直感的にDB操作。
- **Blade テンプレート**: 軽量で安全（自動エスケープ）なビュー。コンポーネント機能あり。
- **Service Container / Service Provider**: 強力なDI（依存注入）コンテナが土台。疎結合・テスト容易。
- **Facade**: `Cache::get()` のように静的呼び出し風にサービスへアクセスする独自の仕組み。
- **Artisan**: コード生成・マイグレーション・キュー実行などのCLI。
- **オールインワン**: ルーティング/ORM/テンプレ/認証/キュー/メール/スケジューラ/ブロードキャストを標準同梱。

## どういう使い方をするのか
- **DB駆動のWebアプリ／SaaS／管理画面／EC／業務システム**。
- **API バックエンド**（Sanctum/Passport、フロントは Vue/React/Inertia）。
- **CLIツール・バッチ・常駐処理**（Artisan command、Queue worker、Scheduler）。

## 強み
- DXが高く、生産性が出る。ドキュメント・エコシステム（Forge/Vapor/Nova/Horizon）が充実。
- 認証・キュー・通知など「よくある要件」が標準で揃う。
- DIコンテナ前提でテスト・拡張がしやすい。

## 弱み・注意点
- 機能が多く「Laravel流のやり方」を覚えるコストがある（Facade/魔法的挙動の学習）。
- Eloquentの手軽さゆえ N+1 やfat modelになりやすい。
- マジックメソッド多用でIDE補完・静的解析に一手間（Laravel IDE Helper / Larastan）。

## エコシステム・周辺ツール
- 認証: Breeze / Jetstream / Fortify / Sanctum（API）/ Passport（OAuth）
- 非同期: Queue（Redis/DB/SQS）＋ **Horizon**（Redisキュー監視）
- 管理/運用: Forge（サーバ）/ Vapor（サーバーレス）/ Nova（管理画面）/ Telescope（デバッグ）
- 高速化: **Octane**（Swoole / RoadRunner で常駐化）
- テスト: Pest / PHPUnit、静的解析: Larastan（PHPStan）

## ひとことメモ（自分の実感）
- （現役として触った所感を後から追記）

## このフォルダの構成
- [laravel11/](./laravel11/) … **Laravel 11 実務リファレンス（フラッグシップ）**。プロジェクトの始め方〜MVC〜DI〜キュー〜テスト〜罠まで、項目=1ファイル。
- （Laravel 12 は 11 の延長。差分は laravel11 の各「この版のポイント」に補記。他バージョンは一旦作らない）
