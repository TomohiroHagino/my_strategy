# Compute Engine（GCP）

## ひとことで言うと
GCPの **IaaS（仮想マシン）** サービス。クラウド上に **VM（仮想マシン）** を立てて、好きなOS・ミドルウェアを動かせる。AWSのEC2に相当する、最も基本的なコンピュートリソース。

## 役割・なぜ必要か
- 「サーバーを1台借りて自由に使う」という、最も柔軟で素朴な選択肢。OS・パッケージ・常駐プロセスを自分で握りたいワークロード（既存アプリの移設、特殊なミドルウェア、GPU計算など）に向く。
- VMのスペックは **マシンタイプ**（vCPUとメモリの組）で決め、起動する中身は **イメージ**（OS入りのテンプレート）で決める。
- ネットワークの出入りは **ファイアウォールルール** で制御する。デフォルトでは多くのポートが閉じており、SSH(22)やHTTP(80)などを明示的に許可する。
- データは **永続ディスク（Persistent Disk）** に置く。VMを止めても消えず、別VMへ付け替えもできる。
- コスト最適化のため、中断され得る代わりに安い **プリエンプティブ / Spot VM** や、台数を自動増減する **マネージドインスタンスグループ（MIG）** が用意されている。

## 基本の使い方（gcloud / コンソール）
```bash
# VMを作成（マシンタイプ・イメージ・ゾーンを指定）
gcloud compute instances create my-vm \
  --zone=asia-northeast1-a \
  --machine-type=e2-small \
  --image-family=debian-12 \
  --image-project=debian-cloud

# SSHで入る／一覧／停止／削除
gcloud compute ssh my-vm --zone=asia-northeast1-a
gcloud compute instances list
gcloud compute instances stop my-vm   --zone=asia-northeast1-a
gcloud compute instances delete my-vm --zone=asia-northeast1-a
```

```bash
# ファイアウォール：特定タグのVMにHTTP(80)を許可
gcloud compute firewall-rules create allow-http \
  --direction=INGRESS --action=ALLOW \
  --rules=tcp:80 --source-ranges=0.0.0.0/0 \
  --target-tags=web

# Spot VM（安価・中断あり）として作成
gcloud compute instances create my-spot-vm \
  --zone=asia-northeast1-a --machine-type=e2-small \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP
```

- **コンソール**では「Compute Engine → VMインスタンス → 作成」でマシンタイプ・リージョン/ゾーン・イメージ・ディスク・ネットワークタグをGUIで設定。
- ファイアウォールは VPC に属し、**ネットワークタグ**（例: `web`）でVMと紐付けると管理しやすい。→ [vpc.md](./vpc.md)

## 実務での勘所
- **ゾーンとリージョン**：リージョン（例: `asia-northeast1`＝東京）の中にゾーン（`-a` 等）がある。VMはゾーン単位。可用性のため複数ゾーンに分散する。
- **マシンタイプは小さく始める**：`e2-micro`/`e2-small` から始め、必要に応じて変更（停止→タイプ変更→起動）。
- バッチや検証など中断OKな処理は **Spot VM** で大幅にコスト削減。
- スケールさせたいWebなら、テンプレート＋ **MIG** で自動回復・オートスケール・ローリング更新を効かせる。
- 起動時の初期設定は **起動スクリプト（startup-script）** で自動化すると再現性が上がる。

## ハマりどころ / アンチパターン
- **ファイアウォールの開けすぎ**：`--source-ranges=0.0.0.0/0` で全開放。SSH(22)を全世界に開けるのは危険。送信元IPを絞るか踏み台/IAP経由に。→ [vpc.md](./vpc.md)
- **停止忘れによる課金**：VMは起動している間ずっと課金される。検証VMの止め忘れに注意。停止してもディスク料金は残る点も忘れずに。
- **ゾーン選択ミス**：作業対象VMと違うゾーンを指定してコマンドが「見つからない」。`instances list` でゾーンを確認。リージョン選択は遅延・データ所在にも効く。
- **Spotを止められない前提で使う**：中断（プリエンプト）され得るので、停止に強い設計（チェックポイント保存等）にしておく。
- **VMに過剰な権限のサービスアカウント**を付ける：最小権限のSAを割り当てる。→ [iam.md](./iam.md)

## 関連
[vpc.md](./vpc.md) / [iam.md](./iam.md)
