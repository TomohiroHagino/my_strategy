# CloudFront / Route53 / ACM（AWS）

## ひとことで言うと
- **CloudFront**：世界中のエッジロケーションにコンテンツをキャッシュして配信する **CDN**。利用者に近い拠点から返すことで速く・安く・安全にする。
- **Route53**：AWS のマネージド **DNS**（ドメイン名 → IP/エンドポイントの名前解決）。ドメイン登録もできる。
- **ACM（AWS Certificate Manager）**：**SSL/TLS 証明書**を無料で発行・自動更新する。HTTPS 化の土台。

## 役割・なぜ必要か
- **CloudFront**：S3 の静的サイトや ALB の動的 API を**エッジでキャッシュ**し、オリジン（S3/ALB）の負荷とレイテンシを下げる。オリジンを隠し、**WAF・DDoS 防御（Shield）**の前段にもなる。
- **Route53**：`example.com` を CloudFront/ALB/S3 へ向ける。**ヘルスチェック付きフェイルオーバー**や**加重・レイテンシベースルーティング**もできる。
- **ACM**：HTTPS は必須要件。ACM なら**証明書の発行と更新を自動化**でき、期限切れ事故を防げる。
- 3つはセットで「**独自ドメイン + HTTPS + CDN 配信**」という定番構成を作る。

## 基本の使い方（CLI / コンソール）
ACM 証明書を発行（CloudFront 用は **us-east-1 必須**）：
```bash
# CloudFront で使う証明書は必ず us-east-1（バージニア北部）で取得する
aws acm request-certificate \
  --domain-name example.com \
  --subject-alternative-names "*.example.com" \
  --validation-method DNS \
  --region us-east-1
# → 表示される CNAME を Route53 に登録して検証（DNS検証）
```
S3 をオリジンにした CloudFront を作る（概略）：
```json
{
  "CallerReference": "my-dist-2026",
  "Origins": { "Quantity": 1, "Items": [{
    "Id": "s3-origin",
    "DomainName": "my-bucket.s3.ap-northeast-1.amazonaws.com",
    "S3OriginConfig": { "OriginAccessIdentity": "" }
  }]},
  "DefaultCacheBehavior": {
    "TargetOriginId": "s3-origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6"
  },
  "Enabled": true
}
```
Route53 で CloudFront へエイリアスを向ける：
```bash
# A レコード（エイリアス）で example.com → CloudFront を指す
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123EXAMPLE \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2",
          "DNSName": "d111111abcdef8.cloudfront.net",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'
```
キャッシュを手動で消す（無効化 / invalidation）：
```bash
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/*"
```

## 実務での勘所
- **TTL 設計が CDN の肝**：ハッシュ付きファイル名（`app.abcd1234.js`）にして**長期キャッシュ（1年）+ HTML だけ短命**にするのが定番。`Cache-Control` をオリジンで正しく付ける。
- **OAC（Origin Access Control）**で S3 を CloudFront 経由のみアクセス可にし、バケットを非公開に保つ。直リンク禁止＝情報漏えい対策。→ [s3.md](./s3.md)
- **Route53 のエイリアスレコード**は CloudFront/ALB/S3 への内部参照で**クエリ課金がなく無料**。CNAME より優先して使う。Apex（`example.com`）には CNAME を張れないのでエイリアス必須。
- 動的 API も CloudFront の背後に置けば**WAF/Shield/HTTP3/圧縮**の恩恵を共通化できる。キャッシュしたくない経路は**キャッシュ無効ポリシー**にする。
- 証明書・ドメイン・配信は**自動更新と監視**を入れておく。期限・配信エラーは CloudWatch で見る。→ [monitoring.md](./monitoring.md)

## ハマりどころ / アンチパターン
- **CloudFront 用 ACM 証明書は us-east-1 でしか使えない**：東京（ap-northeast-1）で取った証明書は CloudFront にアタッチできない。一番多い詰まり所。ALB 用は ALB と同じリージョンで取る、と覚える。
- **キャッシュ無効化（invalidation）を忘れて「直したのに古いまま」**：CloudFront はエッジに残ったまま返す。デプロイ後に `create-invalidation` するか、**ファイル名にハッシュを付けて URL ごと変える**設計にする。`/*` の大量無効化は月1,000パスを超えると課金される点にも注意。
- **DNS 反映待ちを過小評価**：レコード変更は TTL ぶん古い結果が残る。切替直前に TTL を短く（60秒など）しておく。
- **独自ドメイン設定のズレ**：CloudFront の「代替ドメイン名（CNAMEs）」に `example.com` を登録し忘れると `SSL: no alternative certificate subject name matches` 等で 403/SSLエラー。**証明書のドメインと配信の代替ドメイン名と Route53 のレコード、三者を一致**させる。
- **何でもキャッシュして個人情報を配ってしまう**：ログイン後ページを丸ごとキャッシュすると他人のデータが見える事故。`Authorization`/Cookie を含む経路はキャッシュキー設計を慎重に。

## 関連
[s3.md](./s3.md) / [api_gateway.md](./api_gateway.md) / [monitoring.md](./monitoring.md)
