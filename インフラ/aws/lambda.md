# Lambda（AWS）

## ひとことで言うと
**サーバーを意識せずに関数（コード）だけを動かせる**サーバーレス実行環境。サーバーの起動・OS管理・スケーリングをAWSが肩代わりし、こちらは「処理本体（ハンドラ関数）」だけを置く。**イベントが来た時だけ実行**され、実行した分（時間×メモリ）だけ課金される。

## 役割・なぜ必要か
- 常時起動のサーバー（EC2）を持たなくても、「リクエストが来たら／ファイルが置かれたら／時刻になったら」処理を走らせられる。**待機中はコストゼロ**（実行時のみ課金）。
- **イベント駆動**が本質。何かが起きたことを**トリガー**として関数が呼ばれる。代表的なトリガー：
  - **API Gateway** … HTTPリクエスト → 関数（API・Webhookの裏側）。→ [api_gateway.md](./api_gateway.md)
  - **S3** … ファイルアップロード → 関数（画像リサイズ・ETL）。→ [s3.md](./s3.md)
  - **EventBridge** … 定期実行（cron）やAWSイベント → 関数（バッチ・自動化）。
  - 他に SQS / DynamoDB Streams / SNS など。
- スケーリングは自動。同時リクエストが増えれば**実行環境（インスタンス）が並列に増える**。サーバー台数の管理が不要。
- **ランタイム**（実行言語）は Node.js / Python / Java / Go / Ruby / .NET などをマネージドで提供。独自言語は**カスタムランタイム**やコンテナイメージで対応。

## 基本の使い方（CLI / コンソール）
コンソール：Lambda → 「関数の作成」→ ランタイム選択 → インラインエディタ or zip/コンテナでデプロイ → トリガーを追加（API Gateway等）。

CLI（zipでデプロイ＆更新の最小例）：
```bash
# 1) ハンドラ（Python例）を用意し zip 化
cat > handler.py <<'PY'
def handler(event, context):
    name = event.get("name", "world")
    return {"statusCode": 200, "body": f"hello {name}"}
PY
zip function.zip handler.py

# 2) 実行ロール（CloudWatch Logsへの書き込み権限など）を指定して作成
aws lambda create-function \
  --function-name hello \
  --runtime python3.12 \
  --handler handler.handler \
  --role arn:aws:iam::123456789012:role/lambda-basic-exec \
  --zip-file fileb://function.zip \
  --timeout 10 --memory-size 256

# 3) コード更新
aws lambda update-function-code \
  --function-name hello --zip-file fileb://function.zip

# 4) 設定変更（タイムアウト/メモリ/環境変数）
aws lambda update-function-configuration \
  --function-name hello --timeout 30 --memory-size 512 \
  --environment "Variables={STAGE=prod}"

# 5) テスト実行（同期呼び出し）
aws lambda invoke \
  --function-name hello \
  --payload '{"name":"taro"}' --cli-binary-format raw-in-base64-out \
  out.json && cat out.json
```

## 実務での勘所
- **実行ロール（IAM Role）が必須**。関数が他サービス（S3読み取り等）を触るなら、その権限をロールに付ける。最小権限で。→ [iam.md](./iam.md)
- **メモリ設定はCPUも左右する**。メモリを増やすとCPU割り当ても増え、処理が速くなって**結果的に安く速くなる**ことがある。盲目的に最小にしない。
- **ログは CloudWatch Logs** に出る。`print`/`console.log` がそのまま記録される。観測の基本。→ [monitoring.md](./monitoring.md)
- **冪等に作る**。SQS/S3トリガーは**同じイベントが2回届きうる**（at-least-once）。重複実行されても壊れない設計に。
- 状態を持たない（ステートレス）。一時ファイルは `/tmp`（既定512MB、拡張可）に置けるが、実行間で残る保証はない。
- IaC（SAM / CDK / Terraform）で関数・トリガー・権限をまとめて管理するのが実務の定番。→ [iac.md](./iac.md)
- 重い依存ライブラリは**Lambda Layer**で共通化、またはコンテナイメージ方式（最大10GB）にする。

## ハマりどころ / アンチパターン
- **タイムアウト**：既定3秒、**最大15分**。これを超える長時間処理はLambda向きではない（→ Step Functions / ECS / バッチへ）。外部API待ちで詰まると簡単に超える。
- **コールドスタート**：しばらく呼ばれていない関数は、初回に実行環境の用意が入り**初回だけ遅い**。レイテンシ重視のAPIでは Provisioned Concurrency やランタイム選択（軽量なNode/Python）で緩和。
- **VPC接続の初期遅延 / 詰まり**：RDS等のためにVPCへ繋ぐと、ENI割り当て等で起動が遅くなりやすい。さらに**NAT Gateway 経由でないと外部APIに出られない**構成ミスが定番事故。VPC接続は必要な時だけ。
- **デプロイパッケージサイズ**：zip直接アップは50MB、解凍後250MBの上限。超えるならLayer分割やコンテナイメージへ。`node_modules` 肥大に注意。
- **同時実行数の上限**：アカウント既定の同時実行数に達すると**スロットリング**（429）。下流のRDS等を**同時接続で殺す**事故もある（RDS ProxyやReserved Concurrencyで制御）。
- **環境変数に平文で秘密を入れる**：APIキー等は Secrets Manager / SSM Parameter Store から取得する。
- **重い初期化をハンドラ内で毎回やる**：DB接続やSDKクライアントは**ハンドラの外**（グローバル）で初期化し、ウォーム時に再利用する。

## 関連
[api_gateway.md](./api_gateway.md) / [s3.md](./s3.md) / [iam.md](./iam.md) / [monitoring.md](./monitoring.md) / [iac.md](./iac.md)
