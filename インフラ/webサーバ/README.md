# Webサーバ / リバースプロキシ 実務リファレンス（索引）

> アプリ（Rails/Spring/Express…）の**前段に立つ受け口**＝nginx / Apache を中心に、
> 「処理モデル → 各サーバ → リバースプロキシ → アプリサーバ → チューニング/セキュリティ/ログ → 比較」まで詳しくまとめる。

## 全体像
```
 ブラウザ
   │  HTTPS（443）
   ▼
 [ nginx / Apache ]   ← TLS終端・静的配信・振り分け・LB・キャッシュ・レート制限
   │  HTTP / FastCGI（内部・UNIXソケット or TCP）
   ▼
 [ アプリサーバ ]     ← Puma / Unicorn / Gunicorn / Uvicorn / PHP-FPM / Node / Tomcat
   │
   ▼
  DB / Redis
```
- アプリサーバは**動的処理に専念**、静的配信・TLS・大量接続の捌きは前段のWebサーバが担う。役割分担が肝。

## 読む順（おすすめ）
1. [仕組み.md](./仕組み.md) … 処理モデル（プロセス型 vs イベント駆動・C10K・nginxのmaster/worker）
2. [nginx.md](./nginx.md) … nginx 詳細（設定構造・location判定・proxy_passの罠・upstream・実設定）
3. [apache.md](./apache.md) … Apache 詳細（MPM・.htaccess・mod_rewrite・FPM移行）
4. [リバースプロキシ.md](./リバースプロキシ.md) … LBアルゴリズム・TLS終端3方式・L4/L7・タイムアウト・502/503/504
5. [アプリサーバ.md](./アプリサーバ.md) … Webサーバとの違い・Puma/Gunicorn/PHP-FPM/Tomcat・ソケットvsTCP

## 横断トピック
- [チューニング.md](./チューニング.md) … worker/接続/keepalive/バッファ/gzip/proxy_cache/HTTP2・3/OS資源
- [セキュリティ.md](./セキュリティ.md) … TLS設定・セキュリティヘッダ・レート制限・アクセス制御・ハードニング・WAF
- [ログ.md](./ログ.md) … access/error ログ・処理時間フィールド・ローテーション・集約・解析
- [比較.md](./比較.md) … nginx/Apache/Caddy/Envoy/Traefik/HAProxy/クラウドLB の住み分け

## nginx と Apache、どっちを使う？
| 観点 | nginx | Apache |
|---|---|---|
| 処理モデル | **イベント駆動**（少リソースで大量接続） | MPM選択（prefork/worker/event） |
| 得意 | リバースプロキシ・静的配信・高同時接続 | `.htaccess`・mod_php資産・共有ホスティング |
| 設定 | 中央集約（`nginx.conf`） | ディレクトリ単位（`.htaccess`）も可 |
| 現在の立ち位置 | **新規はほぼ nginx**（or Caddy/Envoy） | 既存資産・WordPress系で現役 |

> 新規なら nginx が第一候補。クラウドでは **ALB / CloudFront / Cloud Run** がこの役割を吸収することも多い。→ [比較.md](./比較.md) / [../aws/cloudfront_route53.md](../aws/cloudfront_route53.md)

## このフォルダの構成
```
webサーバ/
├── README.md          ← これ（索引）
├── 仕組み.md          ← 処理モデル（プロセス型/イベント駆動・C10K）
├── nginx.md           ← nginx 詳細
├── apache.md          ← Apache 詳細
├── リバースプロキシ.md ← LB/TLS終端/L4・L7/タイムアウト
├── アプリサーバ.md     ← Puma/Gunicorn/PHP-FPM/Tomcat・ソケットvsTCP
├── チューニング.md     ← 性能設定
├── セキュリティ.md     ← TLS/ヘッダ/レート制限/ハードニング
├── ログ.md            ← ログ設定と読み方
└── 比較.md            ← 他ツールとの住み分け
```

## 各ファイルの書式（テンプレ）
```markdown
# {名前}（Webサーバ）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（設定）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```

## 関連
コンテナ化: [../docker/](../docker/) ／ クラウドのLB/CDN: [../aws/cloudfront_route53.md](../aws/cloudfront_route53.md) ／ コンテナ実行: [../aws/containers.md](../aws/containers.md) ／ 負荷の限界: [../負荷検証/](../負荷検証/) ／ 障害対応: [../障害対応/](../障害対応/)
