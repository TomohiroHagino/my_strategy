# 設定・環境変数（Config / Env）（Gin v1）

## ひとことで言うと
DBのURL・APIキー・ポート番号など、**環境ごとに変わる値をコードの外に出す**仕組み。Goは標準の `os.Getenv` で読めるが、実務では **viper**（設定の一元管理）＋ **godotenv**（`.env` 読み込み）が定番。

## 役割・なぜ必要か
- 開発・ステージング・本番で**接続先や秘密情報が変わる**。コードに直書きすると環境を切り替えられず、secret も漏れる。
- 設定を1か所（struct）に集約すると、起動時に**必須値の欠落を検知**でき、参照も型安全になる。
- 本番は `GIN_MODE=release` でデバッグ出力を止め、ログ量・情報漏れ・性能を最適化する。

## 基本の書き方（コード）
### 標準の os.Getenv（最小構成）
```go
port := os.Getenv("PORT")
if port == "" { port = "8080" } // デフォルト値を必ず用意
r.Run(":" + port)
```

### godotenv：.env を読み込む（開発時）
```go
import "github.com/joho/godotenv"

// main の冒頭で。本番は実環境変数を使うのでエラーは握り潰してよい
_ = godotenv.Load() // カレントの .env を os 環境変数へ展開
```
```bash
# .env（※ .gitignore に必ず入れる）
PORT=8080
DB_DSN=postgres://user:pass@localhost:5432/app
JWT_SECRET=change-me
GIN_MODE=debug
```

### viper：設定を struct にマッピング
```go
import "github.com/spf13/viper"

type Config struct {
    Port      string `mapstructure:"PORT"`
    DBDSN     string `mapstructure:"DB_DSN"`
    JWTSecret string `mapstructure:"JWT_SECRET"`
}

func Load() (Config, error) {
    viper.SetConfigFile(".env")   // .env も読める
    viper.AutomaticEnv()          // OS環境変数を優先的に取り込む
    viper.SetDefault("PORT", "8080") // デフォルト値
    _ = viper.ReadInConfig()      // ファイルが無くてもenvで動かす
    var c Config
    if err := viper.Unmarshal(&c); err != nil { return c, err }
    if c.JWTSecret == "" {        // 必須値の検証（fail fast）
        return c, errors.New("JWT_SECRET is required")
    }
    return c, nil
}
```

### GIN_MODE：本番は release に
```go
// 環境変数 GIN_MODE=release でも切り替わるが、コードからも明示できる
if os.Getenv("APP_ENV") == "production" {
    gin.SetMode(gin.ReleaseMode) // デバッグログ抑制・警告非表示
}
r := gin.Default()
```
```bash
# 本番起動時
GIN_MODE=release ./app
```

## 実務での使い方・定番パターン
- **設定は起動時に1回ロードして struct に詰め**、各層へは引数 or DI で渡す。グローバル参照を散らさない。
- **優先順位**：OS環境変数 > `.env`（`viper.AutomaticEnv()` で env を優先）。コンテナ/CIでは env を直接注入。
- **必須値は起動時に検証**して `log.Fatal`（fail fast）。本番で動き出してから落ちるより安全。
- `.env` はローカル開発専用。本番は **Secret Manager / 環境変数**で注入し、`.env` を配布しない（→ [getting_started.md](./getting_started.md)）。
- デフォルト値は「開発で困らない安全側」に。secret 系にデフォルトを入れない（欠落を検知させる）。

## ハマりどころ / アンチパターン
- **`.env` をコミット**：secret が Git 履歴に永久に残る。`.gitignore` に必須。漏れたら鍵を再生成。`.env.example`（値なしの雛形）だけをコミットする。
- **本番でデバッグモードのまま**：`gin.SetMode(gin.ReleaseMode)` か `GIN_MODE=release` を忘れると、起動時に警告が出続け、デバッグ情報が漏れる。
- **デフォルト値の欠如**：`os.Getenv("PORT")` が空文字のまま `:` で起動して失敗。空判定＋デフォルトを必ず書く。
- **secret にデフォルト値**：`SetDefault("JWT_SECRET", "dev")` のような既定値は本番でそのまま使われる事故の元。必須検証にする。
- **設定をあちこちで `os.Getenv`**：参照が散ると検証漏れ・取り違えが起きる。Config struct に集約する。
- **godotenv の Load エラーで起動停止**：本番に `.env` は無いのが正常。`_ = godotenv.Load()` で握り潰し、実環境変数を使う。

## 関連
[getting_started.md](./getting_started.md)
