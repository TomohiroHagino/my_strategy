# API Gateway（AWS）

## ひとことで言うと
HTTP/HTTPS リクエストを受け取り、**Lambda・HTTP バックエンド・他の AWS サービスへ振り分ける「API の入口（フロントドア）」**。認証・スロットリング・ステージ管理・CORS などを一手に引き受けるマネージドサービス。

## 役割・なぜ必要か
- Lambda はそれ単体では HTTP で叩けない。**API Gateway を前段に置くことで「URL を持った REST/HTTP API」**になり、Web/モバイルから呼べるようになる。
- 認証（誰が呼べるか）・**スロットリング（呼びすぎを止める）**・ログ・メトリクスといった「API の共通の面倒ごと」をアプリ側に書かずに済む。
- ステージ（`dev` / `prod` など）でデプロイ単位を分け、**同じ API を環境別に出し分けられる**。
- 種類が2つある：
  - **REST API**：機能が豊富（リクエスト/レスポンス変換・APIキー・使用量プラン・WAF連携・キャッシュ）。その分**高コスト・設定が複雑**。
  - **HTTP API**：軽量・**低コスト（REST APIの約1/3）**・低レイテンシ。JWT オーソライザー標準対応。込み入った変換は苦手。
  - 迷ったら **まず HTTP API**。細かい制御や API キー課金が要るなら REST API。

## 基本の使い方（CLI / コンソール）
HTTP API を Lambda 統合で作る（CLI）：
```bash
# 1) HTTP API を作成（Lambda を即統合する簡易作成）
aws apigatewayv2 create-api \
  --name my-http-api \
  --protocol-type HTTP \
  --target arn:aws:lambda:ap-northeast-1:123456789012:function:my-fn

# 2) Lambda 側に「API Gateway から呼ばれてよい」権限を付与
aws lambda add-permission \
  --function-name my-fn \
  --statement-id apigw-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com

# 3) ルートとステージを確認（自動作成された $default を使う）
aws apigatewayv2 get-apis
```
JWT オーソライザー（Cognito を認証基盤にする例）：
```yaml
# SAM / IaC でのイメージ（HTTP API）
Auth:
  Authorizers:
    CognitoJwt:
      IdentitySource: "$request.header.Authorization"
      JwtConfiguration:
        issuer: "https://cognito-idp.ap-northeast-1.amazonaws.com/ap-northeast-1_xxxx"
        audience:
          - "my-app-client-id"
  DefaultAuthorizer: CognitoJwt
```
スロットリング（ステージ全体のレート制限）：
```bash
aws apigatewayv2 update-stage \
  --api-id abcde12345 --stage-name '$default' \
  --default-route-settings 'ThrottlingBurstLimit=200,ThrottlingRateLimit=100'
```
コンソールでは「API Gateway → API を作成 → REST/HTTP を選択 → ルート/統合 → オーソライザー → ステージへデプロイ」の順。

## 実務での勘所
- **認証は3択を使い分ける**：
  - **Cognito / JWT オーソライザー**：ユーザーログイン基盤がある SaaS 向け。トークン検証を API Gateway 任せにできる。
  - **Lambda オーソライザー**：独自トークンや外部 IdP、細かい認可ロジックが必要なとき。検証結果は**キャッシュ**してコールを減らす。
  - **API キー + 使用量プラン**（REST API のみ）：B2B でクライアント別にレート/クォータ課金したいとき。
- **ステージ変数**で「ステージごとに向き先 Lambda を変える」など環境差分を吸収する。
- **スロットリング**はアカウント既定（リージョンで10,000 rps 程度）に頼らず、ステージ/ルート単位で明示的に上限を張る。**バックエンド（特にDB）の許容量を超えさせない盾**になる。
- アクセスログ・実行ログは **CloudWatch Logs** へ出す。最初に有効化しておかないと、後から「なぜ403か」が追えない。→ [monitoring.md](./monitoring.md)
- 独自ドメインは **カスタムドメイン名 + ACM 証明書**で割り当てる。CloudFront 経由で配信する構成も多い。→ [cloudfront_route53.md](./cloudfront_route53.md)

## ハマりどころ / アンチパターン
- **REST API と HTTP API を取り違える**：後から乗り換えは実質作り直し。**コスト・必要機能（API キー課金・リクエスト変換・WAF）を最初に確認**して選ぶ。なんとなく REST API を選んで請求が膨らむのが典型。
- **CORS 設定漏れ**：ブラウザから叩くと `No 'Access-Control-Allow-Origin'` で弾かれる。HTTP API は API 側の CORS 設定で、REST API は **OPTIONS メソッド（プリフライト）の応答を自分で用意**する必要があり、ハマりやすい。Lambda プロキシ統合ではレスポンスヘッダに CORS を付ける位置にも注意。
- **タイムアウト29秒の壁**：API Gateway の統合タイムアウトは**最大29秒**。重い処理（PDF生成・大量集計）を同期で待つと `504`。**非同期化（SQS/Step Functions に投げて即時応答）やジョブ化**で回避する。
- **Lambda 権限の付与忘れ**：`AccessDeniedException` や `Internal server error` の正体が「`lambda:InvokeFunction` 未許可」のことが多い。
- **ステージへデプロイし忘れ**：REST API は変更後に「デプロイ」しないと反映されない。設定したのに `403`/旧挙動、は大体これ。
- **オーソライザーのキャッシュ罠**：Lambda オーソライザーの結果キャッシュ（TTL）中はトークンを変えても旧結果が返る。検証中は TTL を 0 にしておく。

## 関連
[lambda.md](./lambda.md) / [cloudfront_route53.md](./cloudfront_route53.md) / [monitoring.md](./monitoring.md)
