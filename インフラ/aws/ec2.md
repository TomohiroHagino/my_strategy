# EC2（AWS）

## ひとことで言うと
**Elastic Compute Cloud** ＝ AWS上で借りる**仮想サーバー（インスタンス）**。OSの入った1台のコンピュータを、必要なスペックで、必要な時間だけ起動して従量課金で使う。

## 役割・なぜ必要か
- 「自前で物理サーバーを買って設置・電源・回線・故障対応…」をせず、**数分でサーバーを1台立てて、不要になれば消せる**ようにするのがEC2。
- Webアプリ、バッチ処理、開発環境など「とにかくOS上で何かを動かしたい」時の基本の置き場。コンテナ（ECS）やサーバーレス（Lambda）の前段にある、最も素朴で自由度の高い選択肢。
- 主要な構成要素：
  - **AMI（Amazon Machine Image）**：OS＋初期ソフトの「型」。これを元にインスタンスを起動する（例：Amazon Linux, Ubuntu）。
  - **インスタンスタイプ**：CPU/メモリの規格（例：`t3.micro`＝小さい/安い、`m5.large`＝汎用、`c5`＝計算重視）。
  - **セキュリティグループ**：インスタンスを守る**仮想ファイアウォール**。どのポートに・どのIPからの通信を許すかを決める。
  - **キーペア**：SSHログイン用の鍵（公開鍵をインスタンスに、秘密鍵を手元に）。
  - **EBS（Elastic Block Store）**：インスタンスに付ける**仮想ディスク**。インスタンスを消してもEBSを残せばデータは残せる。

## 基本の使い方（CLI / コンソール）
コンソールでは EC2 → 「インスタンスを起動」から、AMI・インスタンスタイプ・キーペア・セキュリティグループを選ぶだけで起動できる。CLI例：
```bash
# キーペア作成（秘密鍵を手元に保存。パーミットは厳格に）
aws ec2 create-key-pair --key-name my-key \
  --query 'KeyMaterial' --output text > my-key.pem
chmod 400 my-key.pem

# セキュリティグループ作成 → SSH(22)を「自分のIPだけ」許可（全開放しない）
aws ec2 create-security-group --group-name web-sg --description "web"
aws ec2 authorize-security-group-ingress --group-name web-sg \
  --protocol tcp --port 22 --cidr 203.0.113.10/32   # ← 自分のIP/32

# インスタンス起動（AMI・タイプ・キー・SGを指定）
aws ec2 run-instances --image-id ami-xxxxxxxx \
  --instance-type t3.micro --key-name my-key \
  --security-groups web-sg --count 1

# 一覧 → SSH接続
aws ec2 describe-instances \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,PublicIpAddress]' --output table
ssh -i my-key.pem ec2-user@<PublicIP>

# 停止（課金は止まる/EBSは残る）と 終了（削除）
aws ec2 stop-instances --instance-ids i-xxxx       # stop：また起動できる
aws ec2 terminate-instances --instance-ids i-xxxx  # terminate：消える
```

## 実務での勘所
- **セキュリティグループは「許可リスト」**：既定は全拒否で、開けたものだけ通る。SSH(22)は自分のIP、HTTP/HTTPS(80/443)は必要な範囲、と最小限に。
- **stop と terminate は別物**：`stop`はディスク(EBS)を残して一時停止（課金は止まる、また起動可）。`terminate`は削除（既定でEBSも消える）。消したくないデータはEBSのスナップショットを取る。
- **負荷分散とスケール**：複数インスタンスに振り分けるのが **ELB（ロードバランサー）**、負荷に応じて台数を自動増減するのが **Auto Scaling**。1台で受けず、増減できる構成にしておくと運用が楽。
- **配置はVPCの中**：EC2は必ずVPC（仮想ネットワーク）のサブネット上に置かれる。公開用はパブリックサブネット、DB等は外から触れないプライベートサブネットへ。→ [vpc.md](./vpc.md)
- **権限はIAMロールで渡す**：EC2からS3等を触る時はキーを置かず、インスタンスにロールを付与する。→ [iam.md](./iam.md)
- 用途が「短時間の処理」や「イベント駆動」ならEC2より Lambda/コンテナが向くことも。常時起動が無駄なら見直す。

## ハマりどころ / アンチパターン
- **セキュリティグループで `0.0.0.0/0` を全開放**：SSH(22)やDBポートを全世界に開ける＝即攻撃対象。**送信元IPを `/32` で絞る**。0.0.0.0/0を許すのは原則 80/443 だけ。
- **停止し忘れで課金**：検証で立てたインスタンスを消し忘れ、月末に請求が膨らむ。使い終わったら stop/terminate、Billingアラートも併用。→ [getting_started.md](./getting_started.md)
- **キーペア（.pem）の紛失**：秘密鍵を失うと既存インスタンスにSSHできなくなる。鍵は安全にバックアップ。`chmod 400` を忘れるとSSHが鍵を拒否する。
- **パブリックIPの思い込み**：停止→起動でパブリックIPは変わる（固定したいなら Elastic IP）。
- **1台構成・手動運用のまま本番**：単一障害点になる。ELB＋Auto Scaling、AMI化で再現性を確保する。

## 関連
[vpc.md](./vpc.md) / [iam.md](./iam.md)
