# Volume と Network（Docker）

## ひとことで言うと
**Volume / Bind mount** ＝ コンテナを消してもデータを残すための**永続化の仕組み**（コンテナ内の書き込みは削除で消えるので外に逃がす）。**Network** ＝ コンテナ同士・コンテナと外をつなぐ**通信路**（user-defined bridge ならコンテナ名で名前解決できる）。

## 役割・なぜ必要か
- コンテナの書き込み層は**コンテナと運命を共にする**。`docker rm` でデータも消える。DB のデータ・アップロードファイル・ログなど消えては困るものは、**volume** か **bind mount** でコンテナの外に置く。
- 用途で使い分ける：**named volume** はDocker管理領域に置く永続データ向け（本番DB等）、**bind mount** はホストの実ディレクトリを直結する開発向け（ソース同期）、**tmpfs** はメモリ上の一時データ向け（秘密の一時置き等、ディスクに残さない）。
- コンテナは標準ではネットワークで隔離される。複数コンテナを連携させるには同じ**user-defined bridge ネットワーク**に入れる。すると**コンテナ名/サービス名で相互に名前解決**でき、IP を意識せず `db:5432` のように繋げる。

## 基本の書き方（コード）
Volume / Bind mount / tmpfs：
```bash
# named volume（永続データ。Dockerが管理。消すまで残る）
docker volume create db-data
docker run -d --name db \
  -v db-data:/var/lib/postgresql/data \   # 名前:コンテナ内パス
  postgres:16

# bind mount（ホストの実パスを直結。開発でソース同期）
docker run -d --name web \
  -v "$(pwd)/src:/app/src" \              # ホスト絶対パス:コンテナ内パス
  -p 8080:3000 myapp:1.0

# tmpfs（メモリ上。コンテナ停止で消える。ディスクに残さない）
docker run -d --tmpfs /tmp:rw,size=64m myapp:1.0

# 確認・掃除
docker volume ls
docker volume inspect db-data
docker volume prune          # どのコンテナからも使われていない volume を削除（注意）
```

Network（コンテナ間通信）：
```bash
# user-defined bridge を作る（コンテナ名で名前解決できる）
docker network create appnet

# 同じネットワークに繋ぐと、相手を「コンテナ名」で呼べる
docker run -d --name db    --network appnet postgres:16
docker run -d --name web   --network appnet \
  -e DATABASE_URL=postgres://app@db:5432/appdb \   # ← "db" で解決される
  -p 8080:3000 myapp:1.0

docker network ls
docker network inspect appnet     # 接続中コンテナ・サブネットを確認
```

ネットワークの種類：
```
bridge (既定)      … 1ホスト内の仮想ネット。既定bridgeは名前解決不可、
                     user-defined bridge を作ると名前解決できる
host               … ホストのネットワークを直接使う（ポート分離なし・Linuxのみ）
none               … ネットワーク無し（完全隔離）
```

Compose は上記を自動でやる（→ [compose.md](./compose.md)）：
```yaml
services:
  web:
    build: .
    volumes:
      - ./src:/app/src         # bind mount（開発のソース同期）
  db:
    image: postgres:16
    volumes:
      - db-data:/var/lib/postgresql/data   # named volume（永続化）
volumes:
  db-data:
# networks 省略でも web/db は同じネットに入り "db" で通信できる
```

## 実務での使い方・定番パターン
- **永続データは named volume、開発のソース同期は bind mount**。DBデータを bind mount にすると権限やパフォーマンスで詰まりやすい。DBは named volume が無難。
- **bind mount は開発専用にする**。ホスト依存（パス・権限・OS差）が入るので本番では使わない。本番は named volume か外部ストレージ。
- **コンテナ間は user-defined bridge ＋ 名前で接続**。`docker run --network` か compose を使う。既定 bridge は名前解決できないので避ける。
- **外部公開はポート転送（`-p`）だけに絞る**。DB は `-p` を付けず、同ネット内の app からだけ繋がせる（ホストに晒さない）。
- volume のバックアップは `docker run --rm -v db-data:/data -v $(pwd):/backup alpine tar czf /backup/db.tgz /data` のように一時コンテナ経由で取る。

## ハマりどころ / アンチパターン
- **volume を付け忘れてデータ消失**：DBコンテナを volume 無しで動かし、`rm` で全消し。永続が要るパスは必ず volume に出す。
- **`localhost` でコンテナ間通信しようとする**：コンテナ内の `localhost` は「自分自身」。相手にはサービス名/コンテナ名で繋ぐ。
- **bind mount でホストの `node_modules` が侵入**：`./:/app` を丸ごと bind すると、ホストの（別OS向け）`node_modules` がコンテナの物を上書きして壊れる。匿名volumeで隠す（`-v /app/node_modules`）か、必要なパスだけ bind する。
- **`docker volume prune` / `compose down -v` でデータ全消し**：未使用判定や `-v` で named volume も消える。共有・本番では慎重に。
- **`network: host` の誤用**：ポート分離が無くなりホストのポートと直接衝突する。Linux 限定でもあり、基本は bridge を使う。

## 関連
[compose.md](./compose.md) / [images_containers.md](./images_containers.md) / [commands.md](./commands.md) / [production.md](./production.md) / [pitfalls.md](./pitfalls.md)
