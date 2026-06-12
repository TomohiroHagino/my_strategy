# 実務でハマる罠まとめ（Pitfalls）（CodeIgniter 4）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

## 役割・なぜ必要か
- CI4は「便利な書き方」と「事故る書き方」が紙一重なものが多い（特にRailsと違い**自動エスケープが効かない**点）。症状から該当箇所へ素早く飛ぶための索引。

## モデル / DB
- **`$allowedFields` 未設定で保存されない**：Modelの `insert`/`update` は `$allowedFields` に列挙したカラムだけ通す。空配列だと黙って何も保存されず「保存したのに入らない」。→ [models.md](./models.md)
- **生クエリ `query()` のSQLインジェクション**：`query("... '$x'")` は即SQLi。`query($sql, [$bind])` のバインド配列を使う。Query Builder優先。→ [security.md](./security.md) / [database_query_builder.md](./database_query_builder.md)
- **マイグレーション順**：ファイル名のタイムスタンプ順に実行される。外部キー先のテーブルを後から作ると失敗。依存順に番号を付ける。→ [database_query_builder.md](./database_query_builder.md)
- **`update_all` 相当の一括更新**：`Model` のコールバック/バリデーションを通さない直接Query Builder更新は整合性を自前で担保する。→ [database_query_builder.md](./database_query_builder.md)

## ビュー / セキュリティ
- **`esc()` 忘れ＝XSS**：CI4は自動エスケープ**しない**。素の `<?= $x ?>` はXSS直結。出力は必ず `esc()`、コンテキスト（html/attr/js/url）も正しく選ぶ。→ [security.md](./security.md) / [views.md](./views.md)
- **CSRFフィルタ未有効化**：`Config\Security` を設定しても `Filters.php` の `$globals['before']` に `'csrf'` を入れないと検証ゼロ。フォームのトークンも無意味。→ [security.md](./security.md) / [filters.md](./filters.md)

## ルーティング / 設定
- **自動ルーティングのセキュリティ**：Auto Routing（特に旧来の Legacy）はメソッドが意図せず公開され攻撃面が増える。本番は Defined Routes（明示ルート）推奨、使うなら Auto Routing (Improved)。→ [routing.md](./routing.md)
- **本番 `CI_ENVIRONMENT=production` 忘れ＝情報漏れ**：`development` のままだと詳細エラー・スタックトレース・DB情報が画面に露出。デプロイ時の `.env` を必ず確認。→ [config_env.md](./config_env.md)
- **`.env` の未設定/コミット**：`encryption.key`・DB情報を未設定で起動失敗、または `.env` をコミットして秘密漏洩。gitignore必須。→ [config_env.md](./config_env.md)

## 運用 / 環境
- **`writable` のパーミッション**：`writable/`（ログ・キャッシュ・セッション）に書き込み権限が無いと500。デプロイ時に権限を確認。→ [config_env.md](./config_env.md)
- **`spark` コマンドの環境差**：`php spark migrate` などは実行時の `CI_ENVIRONMENT`・`.env` を見る。本番で叩く前に対象環境・DB接続を確認。→ [config_env.md](./config_env.md) / [database_query_builder.md](./database_query_builder.md)

## テスト
- **テストDB隔離忘れ**：`tests` DBグループや `CI_ENVIRONMENT=testing` を用意せず開発DBへ直接書き込み→データ汚染。トレイト（`FeatureTestTrait`/`DatabaseTestTrait`）と `refresh = true` も忘れず。→ [testing.md](./testing.md)

## 関連
[models.md](./models.md) / [security.md](./security.md) / [views.md](./views.md) / [filters.md](./filters.md) / [routing.md](./routing.md) / [config_env.md](./config_env.md) / [database_query_builder.md](./database_query_builder.md) / [testing.md](./testing.md)
