# 本番運用（Docker）

## ひとことで言うと
コンテナを本番で安定して動かすための運用設定。**再起動ポリシー・リソース上限・ログの集約・レジストリ運用・脆弱性スキャン**を整える。ただし**多数コンテナの自動配置・スケール・自己修復は Docker 単体の範囲外** → オーケストレータ（k8s / ECS）に任せる。

## 役割・なぜ必要か
- 本番では「落ちたら自動で立ち上がる」「暴走しても1コンテナが全リソースを食い尽くさない」「何が起きたかログで追える」が前提。これらは `restart` ポリシー・`--memory`/`--cpus`・ログドライバで担保する。
- イメージは CI でビルドして**レジストリに置き、本番は pull で取得**する。本番ホストでビルドしない（再現性・速度・安全のため）。
- 公開イメージにも脆弱性は混入する。**スキャンを CI に入れて**、危険なイメージのデプロイを止める。
- 単一ホストの compose では、ホスト故障・台数スケール・ローリング更新に耐えられない。本番規模では**オーケストレータ**へ寄せる。

## 基本の書き方（コード）
再起動ポリシー・リソース上限（単一ホスト運用の最低限）：
```bash
docker run -d --name api \
  --restart unless-stopped \      # 異常終了/再起動時に自動復帰（手動stopは復帰しない）
  --memory 512m --memory-swap 512m \  # メモリ上限（OOMで該当コンテナだけ殺す）
  --cpus 1.0 \                    # CPU 1コア相当に制限
  --read-only \                   # ルートFSを読み取り専用（書込は volume/tmpfs のみ）
  --tmpfs /tmp \
  -p 8080:3000 \
  ghcr.io/org/api:1.4.2           # 固定tag（latest不可）
```

compose での本番寄せ（restart / limits / logging / healthcheck）：
```yaml
services:
  api:
    image: ghcr.io/org/api:1.4.2
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 3s
      retries: 3
    logging:
      driver: json-file
      options:
        max-size: "10m"     # ログローテーション（肥大でディスク満杯を防ぐ）
        max-file: "3"
```

レジストリ運用（CIでpush・本番でpull）：
```bash
# CI 側
docker build -t ghcr.io/org/api:1.4.2 .
docker push ghcr.io/org/api:1.4.2
# 本番ホスト側
docker login ghcr.io
docker pull ghcr.io/org/api:1.4.2
docker run -d --restart unless-stopped ... ghcr.io/org/api:1.4.2
```

脆弱性スキャン（CIに組み込む）：
```bash
docker scout cve ghcr.io/org/api:1.4.2     # Docker純正
# または Trivy
trivy image ghcr.io/org/api:1.4.2 --severity HIGH,CRITICAL --exit-code 1
```

## 実務での使い方・定番パターン
- **`--restart unless-stopped`** を基本に。`always` は手動停止も復帰させるので運用で邪魔になりがち。`on-failure:5` は回数制限付き。
- **必ずメモリ/CPU上限を付ける**。1コンテナの暴走でホスト全体が巻き添えになるのを防ぐ。OOM時はそのコンテナだけ殺される。
- **ログは stdout/stderr に出し、ドライバでローテーション or 集約**。`json-file` に `max-size`/`max-file` を必ず設定（無いとログでディスクが満杯になる）。集約は `fluentd`/CloudWatch 等のドライバへ。
- **`HEALTHCHECK` を入れる**。`unhealthy` 判定をオーケストレータが見て差し替えできる。
- **イメージは固定tag/digestで pull**。本番でビルドしない。CI→レジストリ→本番pull の一方向。
- **デプロイは新イメージへ差し替え**（同名コンテナを止めて新tagで起動 or オーケストレータのローリング更新）。コンテナ内を手で直さない。

## ハマりどころ / アンチパターン
- **リソース上限なしで運用**：メモリリークしたコンテナがホストごと道連れにする。`--memory`/`--cpus` は必須。
- **ログ設定なしでディスク満杯**：`json-file` は既定で無制限に溜まる。`max-size`/`max-file` を設定。
- **本番ホストで `docker build`**：依存・キャッシュ・秘密が本番に紛れ再現性が壊れる。CIでビルドしレジストリ経由に。
- **`latest` をデプロイ**：どのバージョンが動いているか不明、ロールバック不能。固定tagで追跡可能に。
- **compose を本番のスケール手段だと思う**：compose は単一ホスト向け。台数スケール・ホスト故障耐性・自己修復は不可。
- **オーケストレーションを自前で組もうとする**：マネージドに寄せる方が安全。→ [../aws/containers.md](../aws/containers.md)（ECS / Fargate / EKS）、GCP なら Cloud Run / GKE。

## 本番のスケールは別レイヤへ
単一ホストを超える運用（多数コンテナの自動配置・スケール・ローリング更新・自己修復）は Docker の外側のオーケストレータが担う：
- **AWS**：ECS / Fargate（サーバ管理不要）/ EKS（k8s）→ [../aws/containers.md](../aws/containers.md)
- **GCP**：Cloud Run（コンテナをサーバーレスに）/ GKE（k8s）
- **Kubernetes**：マルチクラウド/オンプレ横断の標準

## 関連
[best_practices.md](./best_practices.md) / [compose.md](./compose.md) / [commands.md](./commands.md) / [pitfalls.md](./pitfalls.md) / [../aws/containers.md](../aws/containers.md) / [../障害対応/](../障害対応/)
