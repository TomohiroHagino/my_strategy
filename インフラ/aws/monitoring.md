# CloudWatch / 監視・ログ・トレース（AWS）

## ひとことで言うと
**CloudWatch** はAWSの標準監視サービスで、**メトリクス（数値の時系列）・ログ・アラーム・ダッシュボード**を一手に扱う。これに **X-Ray**（分散トレース）と **CloudWatch Logs**（ログ集約）を組み合わせ、「動いているか／どこが遅いか／何が起きたか」を可視化する。

## 役割・なぜ必要か
- 監視がないと **障害に気づくのが顧客より遅れる**。メトリクス＋アラームで「壊れたら即通知」を作る。
- **メトリクス**＝CPU/メモリ/レイテンシ/エラー数などの数値。多くのAWSサービスが自動でCloudWatchへ送る。
- **ログ**＝アプリ/Lambda/コンテナの標準出力やアクセスログ。CloudWatch Logs に集約して検索・分析する。
- **アラーム**＝メトリクスが閾値を超えたら SNS 経由でメール/Slack へ通知、Auto Scaling 発火など。
- **X-Ray**＝1リクエストが複数サービス（APIGW→Lambda→DynamoDB…）を通る経路を追い、**どこで時間を食ったか**を分解する。
- **ダッシュボード**＝主要メトリクスを1画面に並べ、運用の「健康診断ボード」にする。

## 基本の使い方（CLI / コンソール）
```bash
# メトリクスの取得（例: EC2の平均CPU使用率を5分粒度で）
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --start-time 2026-06-12T00:00:00Z --end-time 2026-06-12T01:00:00Z \
  --period 300 --statistics Average

# 自前メトリクス（カスタムメトリクス）を送る
aws cloudwatch put-metric-data \
  --namespace MyApp/Orders --metric-name OrderCount --value 1 --unit Count

# アラーム作成（CPU > 80% が2回連続したらSNSへ通知）
aws cloudwatch put-metric-alarm \
  --alarm-name ec2-cpu-high \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --statistic Average --period 300 --evaluation-periods 2 \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:ap-northeast-1:111122223333:ops-alerts

# ロググループに保持期間を設定（コスト管理の要！既定は「無期限」）
aws logs put-retention-policy --log-group-name /my/app --retention-in-days 30

# Logs Insights でログをクエリ（直近1時間のERRORを集計）
aws logs start-query --log-group-name /my/app \
  --start-time $(date -d '1 hour ago' +%s) --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 50'
```
- コンソールでは **CloudWatch → メトリクス/ログ/アラーム/ダッシュボード** が左メニューに並ぶ。Logs Insights はログ画面の「ログのインサイト」から。
- X-Ray は **CloudWatch → X-Ray トレース**（旧X-Rayコンソール）で、サービスマップとトレース一覧を見る。アプリ側にSDK/エージェント（または ADOT）導入が必要。

## 実務での勘所
- **3つの黄金シグナル**を最初に押さえる: エラー率 / レイテンシ（p50・p95・p99）/ スループット。平均だけ見ると遅い裾を見落とす（**p99を見る**）。
- **構造化ログ**（JSON）で出すと Logs Insights で `filter`・集計が効く。`requestId` を全ログに通すと追跡が一気に楽になる。
- アラームは **SNSトピック1本に集約** → メール・Slack（Chatbot）・PagerDuty へ分岐。アラーム乱立は「狼少年化」して無視される。
- **複合アラーム / 異常検知（Anomaly Detection）** で、固定閾値が難しいメトリクス（季節変動あり）に対応。
- Lambda/コンテナは標準出力が自動でLogsへ。**EC2は CloudWatch Agent を入れないとメモリ/ディスクは取れない**（CPUは取れるがメモリは標準では出ない）。
- ダッシュボードと閾値は **IaCで管理**（手動作成は再現不能になりがち）。→ [iac.md](./iac.md)

## ハマりどころ / アンチパターン
- **ログ保存料金（最大の地雷）**: ロググループの保持期間は**既定で「無期限」**。放置するとログが永遠に貯まり**ストレージ課金が増え続ける**。`put-retention-policy` で 7/30/90日など必ず設定。長期保管は S3 へエクスポート＋ライフサイクルが安い。
- **アラーム閾値の不適切**: 閾値が緩すぎ＝障害に気づけない、厳しすぎ＝誤報連発で無視される。`evaluation-periods`（連続回数）と `period` で「一瞬のスパイク」を除外する設計を。`TREAT_MISSING_DATA`（欠損の扱い）も明示しないと意図せぬ発火/不発火が起きる。
- **メトリクスの粒度と保持期間**: 標準メトリクスは5分粒度、**詳細モニタリング/カスタムは1分粒度（別途課金）**。さらに高解像度は1秒も可。保持は粒度で自動的に粗くなる（例: 1分粒度は15日、5分は63日、1時間は455日で集約）。**古い期間を細かい粒度で見ようとしても残っていない**点に注意。
- **PutMetricData / API 呼び出し課金**: カスタムメトリクスや高頻度の `get-metric-statistics`、Logs Insights クエリは課金対象。メトリクスを1点ずつ大量送信せず**まとめて送る/集計してから送る**。
- **X-Ray のサンプリング**: 全リクエストをトレースすると高コスト＆オーバーヘッド。サンプリングルール（例: 1秒1件＋5%）で間引く。トレース未導入だとサービスマップは空のまま。
- **ログとメトリクスの混同**: 「件数を監視したい」のにログ全文検索で毎回数えるのは高コスト。**メトリクスフィルタ**でログ→メトリクス化し、アラームを張るのが定石。

## 関連
[iac.md](./iac.md) / [getting_started.md](./getting_started.md) / [lambda.md](./lambda.md) / [ec2.md](./ec2.md) / [pitfalls.md](./pitfalls.md)
