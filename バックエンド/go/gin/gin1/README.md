# Gin v1 実務リファレンス（索引）

> **対象 = Gin v1.x（Go 1.21+ 想定）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> Gin は「**ミドルウェアの連鎖を通り、`gin.Context`(`c`) で入出力する**」のが土台。

## リクエストの流れ（全体像）
```
リクエスト → ミドルウェア①(c.Next で次へ) → ② → … → ハンドラ
            各ハンドラ/ミドルウェアは *gin.Context(c) を受け取り、c.JSON 等で返す
            途中で c.Abort すると以降は止まる
```

## 項目（各ファイルへ）

### はじめに / コア
- [getting_started.md](./getting_started.md) … 始め方（go mod / gin.Default / Run）
- [request_flow.md](./request_flow.md) … リクエストの流れ・各層は何を返すか（全体俯瞰）
- [project_structure.md](./project_structure.md) … プロジェクト構成（handler/service/repository）とは
- [routing.md](./routing.md) … ルーティング（GET/POST / パラメータ / グループ）とは
- [middleware.md](./middleware.md) … ミドルウェアとは
- [context.md](./context.md) … `gin.Context`（`c`）とは

### 入出力・検証・エラー
- [request_response.md](./request_response.md) … リクエスト / レスポンスとは
- [binding_validation.md](./binding_validation.md) … バインド / バリデーション（構造体タグ）とは
- [error_handling.md](./error_handling.md) … エラー処理（err値 / Recovery）とは

### データ・認証・設定・運用
- [database.md](./database.md) … DB（GORM / sqlx、内蔵なし）とは
- [auth.md](./auth.md) … 認証（JWT / セッション）とは
- [config_env.md](./config_env.md) … 設定（env / viper）とは
- [testing.md](./testing.md) … テスト（httptest）とは
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Gin v1）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
