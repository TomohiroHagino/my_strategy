# よく使うコマンド早見（Docker）

## ひとことで言うと
日常で打つ Docker コマンドの早見表。**イメージを作る/取る（build/pull）→ 動かす（run）→ 見る（ps/logs/inspect/stats）→ 入る・操作する（exec/cp）→ 止める・消す（stop/rm/rmi/prune）** の流れで覚える。

## 役割・なぜ必要か
- Docker 操作はほぼ固定パターンの繰り返し。「ビルド → 起動 → ログ確認 → 中に入って調査 → 停止・掃除」を素早く回せると、開発・障害調査のテンポが上がる。
- 特に **`logs`（何が起きたか）・`exec`（中に入って確認）・`inspect`（設定の実体）・`stats`（リソース消費）** は障害切り分けの基本道具。
- ディスクは放っておくと停止コンテナ・未使用イメージ・volume で膨らむ。**`prune` 系の掃除コマンド**を知らないと容量不足で詰まる。

## 基本の書き方（コード）
イメージを作る / 取得する：
```bash
docker build -t myapp:1.0 .                 # Dockerfile からビルド（-t でタグ）
docker build -t myapp:1.0 -f Dockerfile.prod .   # ファイル指定
docker pull nginx:1.27                       # レジストリから取得
docker push ghcr.io/org/myapp:1.0            # レジストリへ送る（要 login）
docker images                                # イメージ一覧
docker tag myapp:1.0 org/myapp:1.0           # tag 付け替え
```

起動・確認：
```bash
docker run -d -p 8080:80 --name web nginx:1.27   # detach・ポート転送・命名
docker run -it --rm ubuntu:24.04 bash            # 対話・終了時に自動削除
docker run --env-file .env myapp:1.0             # 環境変数をまとめて渡す
docker run -v data:/var/lib/db postgres:16       # volume マウント

docker ps                                    # 動作中コンテナ
docker ps -a                                 # 停止中も含む全部
docker stats                                 # CPU/メモリ使用量（リアルタイム）
docker top web                               # コンテナ内プロセス一覧
```

ログ・中に入る・調べる・ファイル授受：
```bash
docker logs web                              # 標準出力ログ
docker logs -f --tail 100 web                # 追跡＋直近100行
docker exec -it web sh                        # 動作中コンテナにシェルで入る
docker exec web env                           # コンテナ内で1コマンド実行
docker inspect web                            # 設定の実体（JSON：env/mount/network）
docker inspect -f '{{.State.Status}}' web     # 必要な値だけ抽出
docker cp web:/app/log.txt ./log.txt          # コンテナ→ホストにコピー
docker cp ./conf web:/etc/app/conf            # ホスト→コンテナにコピー
docker diff web                               # 起動後に変更されたファイル
```

止める・消す・掃除：
```bash
docker stop web                              # 停止（SIGTERM→猶予後SIGKILL）
docker start web                             # 再開
docker restart web                           # 再起動
docker rm web                                # コンテナ削除（停止が前提、-f で強制）
docker rmi nginx:1.27                         # イメージ削除

# --- 掃除（ディスク回収）---
docker system df                             # 使用量の内訳（image/container/volume）
docker container prune                        # 停止中コンテナを一括削除
docker image prune                            # ぶら下がり（<none>）イメージ削除
docker image prune -a                         # 未使用イメージを全削除
docker volume prune                           # 未使用 volume 削除（データ注意）
docker system prune                           # 停止コンテナ＋未使用ネット＋ぶら下がりイメージ
docker system prune -a --volumes              # 全部まとめて（破壊的・要確認）
```

Compose（→ [compose.md](./compose.md)）：
```bash
docker compose up -d         # 起動
docker compose ps            # 状態
docker compose logs -f app   # ログ
docker compose exec app sh   # 中に入る
docker compose down          # 停止＋削除（-v で volume も）
```

## 実務での使い方・定番パターン
- **障害調査の定番順**：`docker ps -a`（生死）→ `docker logs`（何が出ているか）→ `docker exec -it ... sh`（中で確認）→ `docker inspect`（設定の実体）→ `docker stats`（リソース）。
- **使い捨て実行は `--rm`**。`docker run -it --rm` でゴミコンテナを残さない。検証や1回限りのスクリプトに便利。
- **`exec` は動作中、`run` は新規**。動いているコンテナを調べるのは `exec`、新しく立てるのは `run`。混同に注意。
- **掃除は範囲を意識**。`prune` は便利だが `-a --volumes` はデータごと消す。共有・本番では `docker system df` で内訳を見てから絞って消す。

## ハマりどころ / アンチパターン
- **`docker rm` できない**：動作中。`docker stop` してから、または `docker rm -f`。
- **`docker system prune -a --volumes` でDBデータ消失**：volume まで巻き込む。本番・共有では使わない。
- **`logs` に何も出ない**：アプリがファイルにログを書いている。Dockerが拾うのは**標準出力/標準エラー**だけ。アプリ側を stdout 出力にする（→ [best_practices.md](./best_practices.md)）。
- **`exec` で `bash` が無い**：軽量イメージ（alpine/slim/distroless）には bash が無い。`sh` を使う、distroless なら `exec` 自体不可。
- **`stop` がすぐ終わらない**：アプリが SIGTERM を無視している。猶予（既定10秒）後 SIGKILL。シグナルを正しく受ける作りにする。

## 関連
[getting_started.md](./getting_started.md) / [images_containers.md](./images_containers.md) / [compose.md](./compose.md) / [volumes_networks.md](./volumes_networks.md) / [pitfalls.md](./pitfalls.md)
