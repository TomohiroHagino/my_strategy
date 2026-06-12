# Dockerfile（Docker）

## ひとことで言うと
イメージの**作り方を1行ずつ書いたレシピ**。`docker build` がこれを上から実行し、**命令ごとに1レイヤ**を積んでイメージを焼く。変わらないレイヤは**キャッシュ再利用**される。ビルド専用の道具を最終イメージに残さない**マルチステージビルド**で、小さく安全なイメージを作る。

## 役割・なぜ必要か
- 「どのベースを使い、何をコピーし、何をインストールし、起動時に何を実行するか」を宣言的に固定する。これがあるから**誰がビルドしても同じイメージ**になる。
- 各命令がレイヤになり、内容が変わらなければキャッシュが効く。**命令の順序**を工夫すると（変わりにくい依存インストールを先、変わりやすいソースコピーを後に）、コード修正のたびに依存を取り直す無駄を消せる。
- ビルドに使うコンパイラや devDependencies は**実行には不要**。マルチステージで「ビルド用ステージ」と「実行用ステージ」を分け、成果物だけ最終イメージにコピーすると、サイズ・攻撃面・脆弱性が一気に減る。

## 基本の書き方（コード）
主要命令の一覧：
```dockerfile
FROM node:20-slim AS base   # 土台イメージ（AS で名前を付けられる）
ARG APP_VERSION=dev         # ビルド時だけの引数（--build-arg で上書き）
ENV NODE_ENV=production     # 実行時にも残る環境変数
WORKDIR /app                # 以降の基準ディレクトリ（無ければ作る）

COPY package*.json ./       # ホスト→イメージへコピー（推奨）
# ADD は URL取得や tar自動展開もできるが、基本は COPY を使う

RUN npm ci                  # ビルド時に実行（レイヤとして焼かれる）

COPY . .                    # 残りのソース
EXPOSE 3000                 # 使用ポートの宣言（公開ではない）
USER node                   # 以降を非rootユーザーで実行（重要）

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:3000/health || exit 1

# CMD は「既定の実行コマンド」。docker run の引数で上書きできる
CMD ["node", "server.js"]
# ENTRYPOINT は「固定の実行体」。run 引数は CMD 側に渡る
# ENTRYPOINT ["node"]
# CMD ["server.js"]   ← ENTRYPOINT と組むと CMD は既定引数になる
```

レイヤキャッシュを効かせる順序（依存を先、コードを後）：
```dockerfile
# ◎ 良い：コードだけ直してもキャッシュで npm ci が再実行されない
COPY package*.json ./
RUN npm ci
COPY . .

# ✗ 悪い：先に全部コピーすると、1行直すたびに npm ci がやり直しになる
# COPY . .
# RUN npm ci
```

マルチステージビルド（ビルド道具を捨て、成果物だけ残す）：
```dockerfile
# --- ステージ1：ビルド（重い道具を使う） ---
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci                  # devDependencies 込み
COPY . .
RUN npm run build           # dist/ を生成

# --- ステージ2：実行（軽い土台。成果物だけ持ち込む） ---
FROM node:20-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev       # 本番依存のみ
COPY --from=build /app/dist ./dist   # ビルド成果物だけコピー
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```
```bash
docker build -t myapp:1.0 .
docker build -t myapp:1.0 --build-arg APP_VERSION=1.0 .   # ARG を渡す
docker build --target build -t myapp:debug .              # 途中ステージだけ作る
```

## 実務での使い方・定番パターン
- **依存インストールを先、ソースコピーを後**。キャッシュ最大化の基本。`COPY package*.json ./` → `RUN install` → `COPY . .`。
- **マルチステージで最終イメージを最小化**。Go/Rust なら最終を `scratch` や `distrochless` に、Node/Python なら `slim`/`alpine` に置く。
- **`CMD` と `ENTRYPOINT` の使い分け**：固定の実行体（必ず動かすバイナリ）は `ENTRYPOINT`、差し替えたい既定引数は `CMD`。CLI 風コンテナは `ENTRYPOINT ["mytool"]`。
- **`ARG` はビルド時、`ENV` は実行時**。秘密は `ARG`/`ENV` に焼かない（履歴に残る）。ビルド時の秘密は BuildKit の `--secret` を使う（→ [best_practices.md](./best_practices.md)）。
- **`HEALTHCHECK` を付ける**。compose や本番オーケストレータが「生きているか」を判定できる（→ [production.md](./production.md)）。

## ハマりどころ / アンチパターン
- **コード修正のたびに依存が再インストール**：`COPY . .` を先に置いているのが原因。依存ファイルだけ先にコピーする。
- **`ENV`/`ARG` に秘密を書く**：`docker history` で見える。イメージ層に焼かれた秘密は消えない。BuildKit secret か実行時注入にする。
- **root で実行**：既定は root。コンテナ脱獄時の被害が大きい。`USER` で非rootにする。
- **`ADD` の多用**：URL取得・tar自動展開で挙動が読みにくい。単純コピーは `COPY`。
- **`apt-get install` のレイヤ肥大**：`RUN apt-get update && apt-get install -y X && rm -rf /var/lib/apt/lists/*` を1行（1レイヤ）にまとめ、キャッシュ残骸を消す。
- **`CMD` をシェル形式で書く**：`CMD node server.js` はシェル経由になり PID 1 がシェルになって SIGTERM が届きにくい。配列形式 `CMD ["node","server.js"]`（exec形式）にする。

## 関連
[images_containers.md](./images_containers.md) / [best_practices.md](./best_practices.md) / [getting_started.md](./getting_started.md) / [compose.md](./compose.md) / [pitfalls.md](./pitfalls.md)
