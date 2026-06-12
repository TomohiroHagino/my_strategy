# Compose（Docker）

## ひとことで言うと
複数コンテナ（app + DB + cache …）の起動定義を**1つの `compose.yaml` にまとめて、`docker compose up` で一括起動**するツール。`docker run` を何個も並べる代わりに、サービス・ポート・環境変数・volume・依存関係を宣言的に書く。**同じcomposeネットワーク内ではサービス名で名前解決**できる。

## 役割・なぜ必要か
- 実アプリは単体では完結しない（Web＋DB＋Redis…）。これらを `docker run` で個別起動・手動接続するのは煩雑で再現性が低い。compose は**起動構成をコード化**して `up` 一発で再現する。
- compose は起動時に**専用ネットワークを自動作成**し、各サービスをそこに繋ぐ。だから `db` というサービスは、app から `db:5432` のように**サービス名でアクセス**できる（IP を気にしなくていい）。
- 開発環境の標準形になる。「リポジトリを clone して `docker compose up` で全部立ち上がる」状態にしておくと、新メンバーのセットアップが激減する。

## 基本の書き方（コード）
```yaml
# compose.yaml （旧名 docker-compose.yml も可）
services:
  app:
    build: .                  # カレントの Dockerfile からビルド（image: 指定なら既存イメージ）
    ports:
      - "8080:3000"           # ホスト8080 → コンテナ3000
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://app:secret@db:5432/appdb  # ← "db" はサービス名
    env_file:
      - .env                  # まとめて読み込む（秘密はこちら、gitignore）
    depends_on:
      db:
        condition: service_healthy   # db が healthy になってから app を起動
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - db-data:/var/lib/postgresql/data   # named volume で永続化
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  db-data:                    # named volume の宣言（消すまで残る）

# networks: は省略すると app/db が同じ既定ネットワークに入り相互通信できる
```

操作：
```bash
docker compose up -d           # ビルド（必要なら）→ 全サービス起動（detach）
docker compose up -d --build   # イメージを強制再ビルドして起動
docker compose ps              # サービスの状態
docker compose logs -f app     # 特定サービスのログ追跡
docker compose exec app sh     # app コンテナの中に入る
docker compose stop            # 停止（コンテナは残す）
docker compose down            # 停止＋コンテナ/ネットワーク削除（volumeは残る）
docker compose down -v         # volume も消す（データ消える・注意）
```

## 実務での使い方・定番パターン
- **`depends_on` だけでは「起動順」しか保証されない**。DB が「接続を受け付ける状態」まで待つには `condition: service_healthy` ＋ `healthcheck` を組む。
- **サービス名で繋ぐ**。app の接続先は `localhost` ではなく `db`（コンテナ間は同ネットワーク内のサービス名）。`localhost:5432` にすると「app コンテナ自身」を指してしまう（→ [volumes_networks.md](./volumes_networks.md)）。
- **秘密は `.env` ＋ `env_file`**。`compose.yaml` に直書きせず `.env` に置き、`.env` は gitignore。サンプルは `.env.example` で共有。
- **開発と本番で compose を分ける**：`compose.yaml`（共通）＋ `compose.override.yaml`（開発用 bind mount 等）。本番は別ファイルを `-f` で指定。
  ```bash
  docker compose -f compose.yaml -f compose.prod.yaml up -d
  ```
- 開発時はソースを **bind mount** してホットリロード（`volumes: - ./src:/app/src`）。本番ではやらない（→ [volumes_networks.md](./volumes_networks.md)）。

## ハマりどころ / アンチパターン
- **app が DB に繋がらない**：接続先を `localhost` にしている。サービス名 `db` を使う。`localhost` はコンテナ自身を指す。
- **`depends_on` で起動したのに DB 未準備で app が落ちる**：起動順≠準備完了。`healthcheck` ＋ `service_healthy` で待つ。
- **`docker compose down -v` でデータ全消し**：`-v` は named volume も削除する。本番・共有環境では絶対に付けない。
- **`compose.yaml` を直したのに反映されない**：イメージがキャッシュされたまま。`--build` を付けて再ビルド。
- **ポート競合（`port is already allocated`）**：ホスト側ポートが他で使用中。`ports` のホスト側番号を変える、または使用中プロセスを止める。
- **本番でこれを使い切ろうとする**：compose は単一ホスト向け。マルチホストの自動配置・スケール・自己修復は範囲外 → [../aws/containers.md](../aws/containers.md)（ECS/EKS）。

## 関連
[volumes_networks.md](./volumes_networks.md) / [dockerfile.md](./dockerfile.md) / [getting_started.md](./getting_started.md) / [production.md](./production.md) / [../aws/containers.md](../aws/containers.md)
