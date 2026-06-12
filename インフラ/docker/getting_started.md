# 始め方（Docker）

## ひとことで言うと
Dockerを使い始めるための最初の一式。**インストール → `docker run hello-world`（疎通確認）→ `Dockerfile` を書く → `build`（イメージ化）→ `run`（コンテナ起動）** までの最小サイクルと、後で散らからないための **フォルダ構成** を整える工程。

## 役割・なぜ必要か
- Dockerの体験は最終的にすべて「**Dockerfile に作り方を書く → build でイメージを焼く → run でコンテナを起動する**」の繰り返しになる。最初にこの一周を手で通すと、以降の概念（レイヤ・volume・compose）がすべてこの流れの上に乗る。
- 「自分のPCには Node がある／ない」「本番は Python 3.11 だがローカルは 3.9」といった**環境差による事故**を、箱（イメージ）に必要物を全部入れることで消す。配るのはイメージ1個、動かすのは `run` 1コマンド。
- フォルダ構成（`Dockerfile` / `.dockerignore` / `compose.yaml` の置き場）を最初に決めておかないと、ビルドが遅くなったり、不要ファイルがイメージに混入したりする。土台を先に作る。

## 基本の書き方（コード）
インストールと疎通確認：
```bash
# インストール（OSごと）
#  - Mac / Windows : Docker Desktop を入れる
#  - Linux         : Docker Engine（docker-ce）+ docker compose plugin を入れる
docker version          # クライアント/サーバ両方の版が出れば起動できている
docker info             # ストレージドライバやリソースの状況

# 疎通確認：イメージを取得→コンテナ実行→メッセージを出して終了
docker run hello-world  # "Hello from Docker!" が出れば成功
```

最小の build → run（Node アプリの例）：
```dockerfile
# Dockerfile（拡張子なし。これが「作り方のレシピ」）
FROM node:20-slim          # 土台イメージ（slim＝小さめ）
WORKDIR /app               # 作業ディレクトリ
COPY package*.json ./      # 依存定義を先にコピー（キャッシュを効かせる）
RUN npm ci                 # ビルド時に依存をインストール
COPY . .                   # 残りのソースをコピー
EXPOSE 3000                # このコンテナが使うポート（あくまで宣言）
CMD ["node", "server.js"]  # コンテナ起動時に実行する既定コマンド
```
```bash
# --- 最小サイクル ---
docker build -t myapp:1.0 .          # カレントの Dockerfile からイメージを焼く
docker images                        # 焼けたイメージを確認
docker run -d -p 8080:3000 --name myapp myapp:1.0
#   -d            : バックグラウンド実行（detach）
#   -p 8080:3000  : ホスト8080 → コンテナ3000 に転送
#   --name myapp  : コンテナに名前を付ける

docker ps                            # 動いているコンテナ
docker logs -f myapp                 # ログを追う
docker exec -it myapp sh             # コンテナの中に入る（シェル）
docker stop myapp && docker rm myapp # 停止して削除
```

## フォルダ構成（始動直後）
小さく始めて、サービスが増えたら `compose.yaml` で束ねる。**【生成】はツール/ビルドが作る（基本さわらない・git管理しない）**、それ以外は自作：
```
my-app/
├── Dockerfile              # イメージの作り方（レシピ）              ★自作
├── .dockerignore           # buildに送らない物の除外リスト           ★自作（重要）
├── compose.yaml            # 複数コンテナの起動定義（app+db 等）      ★自作（旧名 docker-compose.yml）
├── .env                    # composeが読む環境変数の実値            ★自作・秘密を含む→gitignore
│
├── src/                    # アプリのソースコード                   ★自作
│   └── server.js
├── package.json            # 依存定義（言語による）                 ★自作
├── package-lock.json       # 依存のロック（コミットする）           ★自作
│
└── node_modules/           # 【生成】依存の実体 ← .dockerignore で除外、gitignore
```

`.dockerignore` の定番（ビルド高速化と秘密混入防止の要）：
```dockerignore
# build context に送らない＝イメージにも入らない
.git
node_modules
.env
*.log
dist
.DS_Store
# ※ ビルドに不要・巨大・秘密 のものは必ずここで弾く
```

## 実務での使い方・定番パターン
- **`-t name:tag` でタグを必ず付ける**。`myapp:1.0` のように明示する。`latest` 任せは事故のもと（→ [pitfalls.md](./pitfalls.md)）。
- **ポートは `-p ホスト:コンテナ`**。`EXPOSE` は「宣言」であって公開ではない。外から繋ぐには `run -p` が必要。
- 依存インストール（`COPY package*.json` → `RUN npm ci`）を**ソースコピーより先**に置く。コードを直してもキャッシュで依存再取得が省ける（→ [dockerfile.md](./dockerfile.md)）。
- サービスが2つ以上（app + DB など）になったら、`run` を並べず **compose.yaml に移す**（→ [compose.md](./compose.md)）。

## ハマりどころ / アンチパターン
- **`.dockerignore` 未設定**：`node_modules` や `.git`、`.env` まで build context に送られ、ビルドが遅く・イメージが巨大・秘密が混入。最初に必ず置く。
- **`EXPOSE` だけ書いてアクセスできない**：`EXPOSE` は宣言のみ。`docker run -p 8080:3000` が無いとホストから繋がらない。
- **コンテナを消したらデータが消えた**：コンテナの書き込みは削除で消える。DBデータ等は **volume** で永続化する（→ [volumes_networks.md](./volumes_networks.md)）。
- **`latest` で動かして再現しない**：別マシンで `latest` が別バージョンになり挙動が変わる。tag を固定する。

## 関連
[images_containers.md](./images_containers.md) / [dockerfile.md](./dockerfile.md) / [compose.md](./compose.md) / [commands.md](./commands.md)
