# VPC（GCP）

## ひとことで言うと
GCP上に作る**仮想ネットワーク**（Virtual Private Cloud）。VMやCloud SQL等のリソースを置く「私設LAN」にあたり、IPアドレス・経路・通信許可をソフトウェアで定義する。GCPのVPCは**グローバル**（1つのVPCが全リージョンにまたがる）で、その中の**サブネットがリージョン単位**という構造が特徴。

## 役割・なぜ必要か
- リソース同士の**通信境界**を決める。「誰が・どこへ・何のポートで通信できるか」を制御し、内部通信を外から隔離する。
- **グローバルVPC＋リージョナルサブネット**：同じVPCなら東京と大阪のサブネットが内部IPで直接通信でき、リージョンをまたいだ設計がしやすい（AWSのVPCがリージョン単位なのと対照的）。
- 主要な構成要素は次の通り。
  - **ファイアウォールルール**: 通信の許可/拒否（送信元IP・ポート・タグで指定）。
  - **ルート**: パケットの宛先ごとの経路。
  - **Cloud NAT**: 外向きIPを持たないVMがインターネットへ出るための仕組み（インバウンドは開けない）。
  - **限定公開（Private Google Access）**: 外部IPなしのVMから、内部経路でGoogle API（Storage等）へ到達。

## 基本の使い方（gcloud / コンソール）
```bash
# カスタムモードVPC とサブネット（リージョン単位）を作成
gcloud compute networks create my-vpc --subnet-mode=custom
gcloud compute networks subnets create tokyo-subnet \
  --network=my-vpc --region=asia-northeast1 --range=10.0.0.0/24 \
  --enable-private-ip-google-access   # 限定公開アクセスを有効化

# ファイアウォール：社内IPからのSSHだけ許可（ポートを絞る）
gcloud compute firewall-rules create allow-ssh \
  --network=my-vpc --allow=tcp:22 \
  --source-ranges=203.0.113.10/32 --target-tags=ssh-allowed

# Cloud NAT：外部IPなしVMの外向き通信（要 Cloud Router）
gcloud compute routers create my-router \
  --network=my-vpc --region=asia-northeast1
gcloud compute routers nats create my-nat \
  --router=my-router --region=asia-northeast1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```
コンソールなら「VPCネットワーク」からサブネット/ファイアウォール/ルート/Cloud NATを一覧・編集できる。

## 実務での勘所
- **ファイアウォールはタグ/サービスアカウントで束ねる**。個別IPでなく `--target-tags` や SA を対象にすると、VM追加時にルールを書き換えずに済む。
- **`--subnet-mode=custom` を選ぶ**。自動モードは全リージョンにサブネットが勝手に切られるため、本番は手動で必要なリージョンだけ作る。
- **CIDRは将来の拡張を見て設計**。サブネットのIPレンジは後から広げる手間があるので、最初に余裕を持たせる。
- **限定公開（Private Google Access）＋Cloud SQLのプライベートIP**で、外部IPを一切持たない構成にできる。攻撃面を減らせる。→ [cloud_sql.md](./cloud_sql.md)
- **VPCピアリング/共有VPC**で複数プロジェクトのネットワークを繋ぐ設計も覚えておくと、組織規模で効く。

## ハマりどころ / アンチパターン
- **ファイアウォールの開けすぎ（`0.0.0.0/0`）**：SSH(22)やDB(3306/5432)を全開放するのは典型的な事故。送信元を社内IPやIAPに絞る。→ [iam.md](./iam.md)
- **Cloud NATは“ある時点から課金”**：処理データ量＋IP数で料金が発生する。「なぜか外向き通信が高い」の犯人になりがち。不要なら作らない。
- **サブネットのリージョン取り違え**：VMを置きたいリージョンにサブネットが無く起動できない。VPCはグローバルでも**サブネットはリージョン固定**を忘れない。
- **暗黙の許可ルールへの誤解**：同一VPC内も既定で全通する訳ではない。内部通信を期待するなら明示的にallowを書く（既定はingress全拒否）。
- **外部IPを付けたまま放置**：不要な外部IPは課金とリスク。Cloud NAT＋限定公開で内部に閉じる設計を優先。→ [compute_engine.md](./compute_engine.md)

## 関連
[compute_engine.md](./compute_engine.md) / [cloud_sql.md](./cloud_sql.md) / [iam.md](./iam.md)
