# Hono

## 一言で
**超高速・軽量・TypeScriptファースト**のWebフレームワーク。Web標準（`Request`/`Response`）の上に作られ、**Cloudflare Workers / Deno / Bun / Node.js / Vercel / AWS Lambda など複数ランタイムで同じコードが動く**のが最大の特徴。Edge / サーバーレス時代の新定番。

## 特徴
- **マルチランタイム**：1つのコードが Workers でも Bun でも Node でも動く（アダプタで吸収）。
- **爆速ルーター**：`RegExpRouter` 等で高速マッチ。極小バンドル。
- **Web標準ベース**：`Request`/`Response` をそのまま使う（Node固有APIに依存しない）。
- **Context オブジェクト `c` 中心**：`c.req`（入力）と `c.json()/c.text()`（出力）を `c` 1つで扱う。
- **型安全 RPC**：サーバの型をクライアントと共有し、APIを型付きで呼べる（Hono の目玉）。
- **組込ミドルウェアが充実**：cors / logger / jwt / cache / secureHeaders など同梱。

## どういう使い方をするのか
- **Edge / サーバーレス API**（Cloudflare Workers・Vercel・Lambda）。
- 軽量な **REST API / BFF**、Bun/Deno アプリ。
- 速度・起動の軽さ・型安全が効く場面。

## 強み / 弱み
- 強み：速い・軽い・どこでも動く・TS/型RPCでDXが高い。
- 弱み：フルスタックではない（API向き）。エコシステムは Express より新しい（歴史が浅い）。

## Express との違い（ざっくり）
| | Express | Hono |
|---|---|---|
| 前提 | Node中心 | **マルチランタイム**（Workers/Deno/Bun/Node） |
| 言語 | JS寄り | **TypeScriptファースト**（型RPC） |
| 速度/サイズ | 普通 | **高速・極小** |
| 成熟度 | 枯れて巨大資産 | 新しい・伸び盛り |
| 入出力 | `req`/`res` | Web標準 ＋ Context `c` |

## このフォルダの構成
- [hono4/](./hono4/) … **Hono v4 実務リファレンス（フラッグシップ）**。始め方〜Context〜ミドルウェア〜型RPC〜マルチランタイム〜テスト〜罠まで、項目=1ファイル。

> 関連: 同じNode上の定番は [../express/](../express/)。Nodeの土台は [../README.md](../README.md)。
