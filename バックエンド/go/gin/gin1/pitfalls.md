# 実務でハマる罠まとめ（Pitfalls）（Gin v1）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

## 役割・なぜ必要か
- Gin は薄いフレームワークで、「素朴に書くと事故る」箇所が Go の流儀（明示的エラー処理・並行性）と絡んで多い。症状から該当箇所へ素早く飛ぶための索引。

## エラー処理 / 制御フロー
- **`if err != nil` の徹底**：戻り値の `err` を無視すると失敗が黙って進む。受け取ったら必ず分岐・ログ・適切なステータス返却を。→ [error_handling.md](./error_handling.md)
- **panic は Recovery で受ける**：ハンドラ内 panic はプロセスを落としかねない。`gin.Default()`（Recovery入り）か自前 Recovery ミドルウェアで500に変換。→ [error_handling.md](./error_handling.md)

## ミドルウェア
- **`c.Abort()` しないと続行する**：ミドルウェアで `c.JSON(401, ...)` を返しても `c.Abort()` を呼ばないと後続ハンドラが実行される。中断は `c.AbortWithStatusJSON` 等で。→ [middleware.md](./middleware.md)

## バインド / バリデーション
- **`ShouldBind` と `Bind` の違い**：`Bind` は失敗時に自動で400を書き込む（以降の応答制御が崩れやすい）。`ShouldBind` は err を返すだけで応答は自分で制御。基本 `ShouldBind` 系を使う。→ [binding_validation.md](./binding_validation.md)

## データベース
- **DBは内蔵なし**：Gin に ORM/DB層は無い。GORM や sqlx を自分で組み込み、接続・マイグレーション・トランザクションを管理する。→ [database.md](./database.md)

## 並行処理 / Context
- **goroutine には `c.Copy()`**：リクエスト処理後に `*gin.Context` は再利用/無効化される。非同期処理へ渡すなら `cCp := c.Copy()` のコピーを使う。元の `c` を goroutine で触ると競合/パニック。→ [context.md](./context.md)

## 起動 / 構成
- **`gin.Default()` と `gin.New()`**：`Default` は Logger + Recovery 入り。`New` は素のエンジン（ミドルウェア無し）。必要なものを把握して選ぶ。→ [getting_started.md](./getting_started.md)
- **無規約で構造が肥大**：全部 main.go に書くと handler/service/repository が混ざり破綻。レイヤを分けて責務を切る。→ [project_structure.md](./project_structure.md)

## 認証・認可
- **認可漏れ（IDOR）**：認証だけ通して本人/権限チェックを忘れると他人のリソースを操作できる。`user_id` スコープ＋所有者/権限判定を必ず。→ [auth.md](./auth.md)

## 設定 / 環境
- **`.env` をコミット**：secret が Git 履歴に残る。`.gitignore` 必須・漏れたら再生成。雛形は `.env.example` を置く。→ [config_env.md](./config_env.md)
- **本番が release モードでない**：`gin.SetMode(gin.ReleaseMode)` / `GIN_MODE=release` を忘れるとデバッグ情報漏れ・警告継続。本番は必ず release。→ [config_env.md](./config_env.md)

## 関連
[error_handling.md](./error_handling.md) / [middleware.md](./middleware.md) / [binding_validation.md](./binding_validation.md) / [database.md](./database.md) / [context.md](./context.md) / [auth.md](./auth.md) / [config_env.md](./config_env.md)
