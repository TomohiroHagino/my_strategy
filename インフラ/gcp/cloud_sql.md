# Cloud SQL（GCP）

## ひとことで言うと
Googleが運用を肩代わりしてくれる**フルマネージドのRDB（リレーショナルDB）**サービス。中身は **MySQL / PostgreSQL / SQL Server** の3エンジンから選べ、自前でVM上にDBを立てるのに比べ「パッチ当て・バックアップ・冗長化・監視」を任せられる。

## 役割・なぜ必要か
- アプリには結局「トランザクション・JOIN・一意制約が効く普通のSQL DB」が要る場面が多い。Cloud SQLはそれを**運用ごと**引き受ける。
- 自前運用（VM＋DBインストール）と比べた価値は次の3つ。
  - **高可用性（HA）**: リージョン内の別ゾーンにスタンバイを持ち、障害時に自動フェイルオーバー。
  - **リードレプリカ**: 読み取り負荷を別インスタンスへ逃がしてスケールさせる。
  - **自動バックアップ / PITR（ポイントインタイムリカバリ）**: 日次バックアップ＋binlog/WALで任意時点へ復旧。
- つまり「DBは欲しいが、DBAの運用は持ちたくない」をGCPで埋めるのがこのサービス。

## 基本の使い方（gcloud / コンソール）
```bash
# PostgreSQL インスタンス作成（HA構成 = REGIONAL）
gcloud sql instances create my-pg \
  --database-version=POSTGRES_16 \
  --tier=db-custom-2-7680 \
  --region=asia-northeast1 \
  --availability-type=REGIONAL \
  --storage-auto-increase

# DBとユーザを作る
gcloud sql databases create appdb --instance=my-pg
gcloud sql users create appuser --instance=my-pg --password='****'

# リードレプリカを追加（読み取り負荷の分散）
gcloud sql instances create my-pg-replica \
  --master-instance-name=my-pg --region=asia-northeast1

# 接続名（project:region:instance）を確認 ← Proxyや各種接続で使う
gcloud sql instances describe my-pg --format='value(connectionName)'
```

```bash
# Cloud SQL Auth Proxy で安全に接続（IAM認証 + TLS、IP直叩き不要）
./cloud-sql-proxy --port 5432 my-project:asia-northeast1:my-pg
# 別端末から localhost:5432 へ普通に psql できる
psql "host=127.0.0.1 port=5432 dbname=appdb user=appuser"
```
コンソールなら「SQL → インスタンスを作成」から同じ設定（エディション/エンジン/可用性/接続）をGUIで選べる。

## 実務での勘所
- **接続は Cloud SQL Auth Proxy 経由が基本**。パスワードに加えてIAMで認可し、通信はTLSで暗号化、IP許可リストの運用から解放される。
- **HAは“課金が約2倍”**。本番はREGIONAL、開発・検証はZONAL（単一ゾーン）でコストを抑える、と割り切る。
- **リードレプリカは書き込みできない**。アプリ側で「書きはプライマリ / 読みはレプリカ」のルーティングを実装する前提。
- **接続数の上限**に注意。サーバーレス（Cloud Run）から大量インスタンスが繋ぐとコネクション枯渇しやすい → プール（PgBouncer等）やプール設定で抑える。
- **メンテナンスウィンドウ**を業務影響の少ない時間帯に固定しておく。再起動を伴う更新がそこに寄る。

## ハマりどころ / アンチパターン
- **パブリックIPを有効化して `0.0.0.0/0` で開ける**＝事実上インターネット公開。本番は**プライベートIP（VPC内）＋Auth Proxy**にする。→ [vpc.md](./vpc.md)
- **インスタンスサイズの過大/過小**：vCPU・メモリで料金が決まる。盛りすぎると無駄、足りないと接続・クエリで詰まる。まず小さく始めてメトリクスで上げる。
- **ストレージは増やせるが減らせない**（自動増加は片道）。`storage-auto-increase` の暴走で課金が膨らむことがある。
- **Cloud Runからの接続設定漏れ**：`--add-cloudsql-instances` を付け忘れ＆Proxy未設定で繋がらない。Cloud Run側に接続名を渡す or サイドカーProxyを置く。→ [cloud_run.md](./cloud_run.md)
- **バックアップ無効のまま運用**：作成時にデフォルトでも、明示的に自動バックアップ＋PITRを有効化したか確認する。

## 関連
[cloud_run.md](./cloud_run.md) / [firestore.md](./firestore.md) / [vpc.md](./vpc.md)
