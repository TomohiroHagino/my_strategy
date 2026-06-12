# PHP

## 一言で
Web開発で最も広く使われてきたサーバサイドのスクリプト言語。動的型付けだが、8.x系で型・パフォーマンス・言語機能が大幅強化され「モダンな言語」に生まれ変わった。

## 特徴
- **Web特化の出自**: HTMLに埋め込めるテンプレート起源。リクエストごとにスクリプトが実行される「shared-nothing」モデル（状態を持ち越さない＝スケールしやすい）。
- **8.x でモダン化**: JITコンパイル、`enum`、`readonly` プロパティ、名前付き引数、`match`式、Attributes（アノテーション）、Constructor Property Promotion、`never`/union型など。
- **動的型＋漸進的型付け**: 型宣言は任意だが、付ければ静的解析（PHPStan/Psalm）で安全性を上げられる。
- **巨大なエコシステム**: Composer（パッケージ管理）、PSR（標準仕様）、Laravel / Symfony。

## どういう使い方をするのか
- **Webアプリ／API**（Laravel・Symfony が主役）。
- **CMS・既存資産**: WordPress をはじめ世界のWebの大きな割合がPHP。
- **CLIスクリプト・バッチ**（artisan / symfony console）。

## 強み
- 学習しやすく情報が多い。レンタルサーバ〜クラウドまで動く先が広い。
- shared-nothing で水平スケールが素直。
- 8.x で型・速度が実用十分。Composer/PSRで近代的な開発ができる。

## 弱み・注意点
- 言語の歴史的経緯で「古い書き方／悪い記事」も多い（情報の取捨選択が要る）。
- 長時間プロセス・常駐型は不得手（→ Swoole / Laravel Octane で補う）。
- 動的型ゆえ大規模では静的解析（PHPStan）併用が前提。

## 向いている場面 / 向かない場面
- 向いている: Webアプリ/API全般、素早い立ち上げ、広いホスティング要件。
- 向かない: CPUバウンドな重い計算、常駐・低レイテンシなリアルタイム基盤（専用言語向き）。

## エコシステム・周辺ツール
- パッケージ: Composer / Packagist
- 標準仕様: PSR（PSR-4オートロード等）
- 品質: PHPStan / Psalm（静的解析）、PHP CS Fixer / Pint（整形）
- テスト: PHPUnit / Pest
- フレームワーク: **Laravel**（→ [laravel/](./laravel/)）/ Symfony
- 実行: PHP-FPM ＋ nginx、高速化に Laravel Octane（Swoole / RoadRunner）

## ひとことメモ（自分の実感）
- （現役として触った所感を後から追記）

## このフォルダの構成
- [laravel/](./laravel/) … Laravel（フレームワーク）。フラッグシップの版別リファレンス。
  - （PHP言語そのものの「○○とは」深掘りも、今後この `php/` 直下に切り出していける）
