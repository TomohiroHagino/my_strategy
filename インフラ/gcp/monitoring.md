# Cloud Monitoring / Logging（GCP）

## ひとことで言うと
**Cloud Monitoring**＝メトリクス（CPU・メモリ・レイテンシ等の数値）を集めてダッシュボード表示・アラートする監視サービス。**Cloud Logging**＝アプリやGCPサービスのログを一箇所に集約し、**Log Explorer** で検索・分析するログ基盤。合わせてGCPの運用監視の中核（旧称 Stackdriver）。

## 役割・なぜ必要か
- 動いているシステムの「**今どうなっているか（メトリクス）**」と「**何が起きたか（ログ）**」を可視化し、異常を**人が気づく前にアラート**で知らせるため。
- GCPの多くのサービス（Cloud Run / GKE / Compute Engine 等）は**標準でメトリクス・ログを自動送信**するので、設定ゼロでも基本の監視が始められる。
- 関連サービスがセットで揃う：
  - **Cloud Monitoring** … 数値の監視・ダッシュボード・アラートポリシー。
  - **Cloud Logging** … ログの集約・検索（Log Explorer）・保持・転送（Sink）。
  - **Error Reporting** … アプリの例外を自動でグルーピングし「同じエラーが何件・いつから」を集計。
  - **Cloud Trace** … リクエストがどのサービスで何ms掛かったか（分散トレーシング）を可視化し、遅延の犯人を特定。

## 基本の使い方（gcloud / コンソール / クエリ）
```bash
# ログを書く（アプリから手動で送る例）。重要度(severity)を付けると検索/アラートしやすい
gcloud logging write my-app-log \
  '{"message":"checkout failed","order_id":123}' \
  --severity=ERROR --payload-type=json

# 最近のERRORログを読む
gcloud logging read \
  'resource.type="cloud_run_revision" AND severity>=ERROR' \
  --limit=20 --freshness=1h
```

```text
# Log Explorer のクエリ言語（コンソールで使う）
# 例: 特定サービスの 500 エラーだけ絞り込む
resource.type = "cloud_run_revision"
resource.labels.service_name = "checkout"
severity >= ERROR
httpRequest.status = 500
```

```bash
# ログベースメトリクス: 「ERRORログの件数」を数値メトリクス化する
gcloud logging metrics create checkout_errors \
  --description="checkout ERROR count" \
  --log-filter='resource.labels.service_name="checkout" AND severity>=ERROR'
```

コンソールでは Monitoring の「Dashboards」でグラフを並べ、「Alerting」でアラートポリシー（条件＋通知先）を作る。通知先（Notification Channel）はメール・Slack・PagerDuty 等を登録できる。アラートはコード管理したいので [iac.md](./iac.md)（Terraform）で定義するのが定番。

## 実務での勘所
- **SLI/SLO を決めてからアラート**：監視すべき指標（エラー率・p95レイテンシ・可用性）を先に決め、それを割ったら鳴らす。むやみに全メトリクスにアラートを張らない。
- **ログには severity と構造化（JSON）を**：`severity=ERROR` や構造化フィールドを付けると、検索・ログベースメトリクス・アラートが一気にやりやすくなる。
- **ログベースメトリクス**で「特定エラーの発生回数」を数値化し、それにアラートを掛けると「テキストログ→数値監視」に橋渡しできる。
- **Error Reporting** を有効にして例外を集約し、新種エラー発生時に通知。**Trace** で遅いエンドポイントの内訳を見て改善。
- **通知の多重化と当番**：重要アラートは複数チャネル（メール＋Slack）へ。鳴りっぱなしを防ぐため閾値と継続時間（例「5分続いたら」）を調整。
- 監視設定も [iac.md](./iac.md) でコード化し、環境ごとに同じアラートを再現する。

## ハマりどころ / アンチパターン
- **ログのストレージ課金を見落とす**：ログは取り込み量と保持期間で課金される。デバッグログ（`DEBUG`/`INFO`）を本番で大量に出し続けると地味に高額。**除外フィルタ（Exclusion）**で不要なログを取り込み前に捨て、**保持期間**を要件に合わせて短くする。
- **何でもアラート（アラート疲れ）**：些細なスパイクで鳴らすと無視する文化ができ、本当の障害を見逃す。継続時間条件や閾値で「本当にまずい時だけ」鳴らす。
- **ログベースメトリクスの取り過ぎ**：フィルタが広すぎると大量のログにマッチして課金・処理が増える。対象を絞ったフィルタにする。
- **アラートポリシーの通知先未設定**：条件は作ったが Notification Channel を紐付け忘れ、鳴っても誰にも届かない。作成時に通知先まで必ず設定する。
- **権限不足でログが出ない／見えない**：アプリのサービスアカウントに `logging.logWriter`、閲覧者に `logging.viewer` が無いと書けない・読めない。IAM設定を忘れない。
- **メトリクスのラベル爆発**：ユーザーIDなど高カーディナリティな値をラベルに入れると時系列が無数に増えてコスト増・表示崩れ。ラベルは低カーディナリティに保つ。

## 関連
[iac.md](./iac.md)
