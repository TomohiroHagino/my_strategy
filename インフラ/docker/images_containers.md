# イメージとコンテナ（Docker）

## ひとことで言うと
**イメージ** ＝ アプリ実行に必要な物を固めた**読み取り専用のテンプレート**（レイヤの積み重ね）。**コンテナ** ＝ そのイメージを起動した**実行中のインスタンス**。`Dockerfile`→`build`→**image**→`run`→**container** の関係で、1つのイメージから何個でもコンテナを作れる。

## 役割・なぜ必要か
- イメージは「クラス」、コンテナは「インスタンス」に近い。**配布・保存するのはイメージ**（不変）、**動かすのはコンテナ**（使い捨て）。この分離が再現性の核心：同じイメージなら誰のマシンでも同じ中身が動く。
- イメージは**レイヤの積層**でできている。`Dockerfile` の命令1つ＝1レイヤ。レイヤは内容のハッシュで識別され、**変わらないレイヤは再利用**される。だからビルドが速く、複数イメージでベース層が共有されディスクも節約できる。
- コンテナはイメージの上に**薄い書き込み可能レイヤ**を1枚乗せて動く。コンテナ内の変更はこの層に溜まり、**コンテナを消すと一緒に消える**。永続化したいデータは別途 volume に逃がす（→ [volumes_networks.md](./volumes_networks.md)）。

## 基本の書き方（コード）
```bash
# イメージを取得する（レジストリから pull）
docker pull nginx:1.27            # tag を指定（latest 任せにしない）

# イメージ一覧（ID・サイズ・tag）
docker images

# イメージからコンテナを複数起動できる（同じテンプレ→別インスタンス）
docker run -d --name web1 nginx:1.27
docker run -d --name web2 nginx:1.27   # web1 とは独立した別コンテナ

# コンテナ一覧（-a で停止中も）
docker ps -a

# レイヤ構成を見る（どの命令でレイヤが増えたか）
docker history nginx:1.27

# イメージ／コンテナの詳細（環境変数・マウント・ネットワーク等）
docker inspect nginx:1.27
docker inspect web1
```

tag とレジストリ（保管・配布）：
```bash
# tag の形： [レジストリ/][名前空間/]リポジトリ:タグ
#   例) nginx:1.27
#       ghcr.io/myorg/myapp:1.2.3
#       123456789.dkr.ecr.ap-northeast-1.amazonaws.com/myapp:1.0

# ローカルイメージに別tagを付ける（push用に付け替え）
docker tag myapp:1.0 ghcr.io/myorg/myapp:1.0

# レジストリにログインして push（配布）
docker login ghcr.io
docker push ghcr.io/myorg/myapp:1.0
```

## 実務での使い方・定番パターン
- **イメージは不変、コンテナは使い捨て**として扱う。設定変更はコンテナを直接いじらず、Dockerfile を直して再ビルド→新コンテナに差し替える（Immutable Infrastructure）。
- **tag は意味のある値を明示**。`v1.2.3`（SemVer）や Git の commit SHA を使う。`latest` は「最新」を保証せず、別マシンで中身が食い違う原因になる（→ [pitfalls.md](./pitfalls.md)）。
- **レジストリは用途で選ぶ**：公開は Docker Hub / GitHub Container Registry（ghcr.io）、AWS 上なら ECR、GCP なら Artifact Registry。CI でビルドして push、本番で pull、が定番。
- 不要イメージ・停止コンテナはディスクを食う。`docker system df` で使用量を確認し `prune` で掃除する（→ [commands.md](./commands.md)）。

## ハマりどころ / アンチパターン
- **コンテナに直接ログインして手で設定変更**：再ビルドすると消える「秘伝のタレ」になる。変更は Dockerfile に書く。
- **`latest` を本番で使用**：pull するタイミングで中身が変わり再現しない。固定tag（commit SHA 等）を使う。
- **同名コンテナで `run` できない**：`--name web1` が既存だと衝突する。`docker rm web1` してから、または別名で。
- **イメージ削除できない（`image is being used`）**：そのイメージから作ったコンテナが残っている。先にコンテナを `rm`、それから `rmi`。
- **ビルドのたびにイメージが増殖**：古い無名イメージ（`<none>`）が溜まる。`docker image prune` で掃除。

## 関連
[dockerfile.md](./dockerfile.md) / [getting_started.md](./getting_started.md) / [volumes_networks.md](./volumes_networks.md) / [commands.md](./commands.md) / [../aws/containers.md](../aws/containers.md)
