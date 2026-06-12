# nginx（Webサーバ）

## ひとことで言うと
**イベント駆動で大量の同時接続を少ないリソースで捌く**Webサーバ兼リバースプロキシ。いまの新規構成では「アプリの前段」の事実上の標準。静的ファイル配信・TLS終端・複数アプリへの振り分け・ロードバランス・キャッシュ・レート制限をこなす。

## 役割・なぜ必要か
- **静的配信**: 画像/CSS/JS を `sendfile` でカーネル内転送し高速に返す。
- **リバースプロキシ**: 受けたリクエストを裏のアプリ（Puma/Gunicorn/PHP-FPM/Node…）へ取り次ぐ。→ [リバースプロキシ.md](./リバースプロキシ.md) / [アプリサーバ.md](./アプリサーバ.md)
- **TLS終端**: HTTPS復号をここで終え、内部はHTTPで軽くする。→ [セキュリティ.md](./セキュリティ.md)
- **ロードバランス**: 複数アプリインスタンスへ振り分け（`upstream`）。
- 「接続ごとにプロセス」のApache preforkと違い、**1ワーカーで数千〜数万接続**をepollループで扱える。→ [仕組み.md](./仕組み.md)

## 設定ファイルの構造（コンテキスト階層）
nginxの設定は**入れ子のブロック（コンテキスト）**で、内側は外側を継承する。
```nginx
# main コンテキスト（最上位）
user  nginx;
worker_processes auto;          # = CPUコア数
error_log /var/log/nginx/error.log warn;

events {                        # 接続処理の設定
    worker_connections 1024;    # 1ワーカーあたり最大接続
}

http {                          # HTTP全体の設定（ここが本体）
    include       mime.types;
    sendfile      on;
    keepalive_timeout 65;
    gzip on;

    upstream app { server 127.0.0.1:3000; }   # 裏のアプリ群

    server {                    # バーチャルホスト（1サイト）
        listen 443 ssl;
        server_name example.com;

        location / {            # パスごとの処理
            proxy_pass http://app;
        }
    }
}
```
- 主なディレクティブの置き場所：`worker_*`→main、`worker_connections`→events、`gzip`/`upstream`→http、`listen`/`server_name`→server、`proxy_pass`/`try_files`→location。
- **継承**：`http`に書いた`gzip on`は全`server`/`location`に効く。内側で再宣言すると上書き。

## location のマッチング（最重要・事故多発）
`location` は書いた順ではなく**修飾子の優先順位**で決まる。
| 記法 | 種類 | 優先 |
|---|---|---|
| `location = /path` | 完全一致 | **1（最優先）** |
| `location ^~ /path` | 前方一致（正規表現を抑止） | 2 |
| `location ~ /regex` | 正規表現（大小区別あり） | 3（記述順で最初の一致） |
| `location ~* /regex` | 正規表現（大小無視） | 3 |
| `location /path` | 前方一致（最長一致が勝つ） | 4（最後） |
```nginx
location = /favicon.ico { ... }          # 完全一致で即決
location ^~ /assets/    { ... }          # /assets/ 配下は正規表現に渡さない
location ~* \.(jpg|css|js)$ { ... }      # 拡張子で一致（正規表現）
location /            { proxy_pass http://app; }   # 上に当たらなければここ
```
- **罠**：`location ~ \.php$` と `location /` の関係、`^~` を付け忘れて静的が正規表現側に吸われる、など。優先順位を必ず意識する。

## proxy_pass の末尾スラッシュ（最大級の罠）
`proxy_pass` の URL に**末尾 `/` があるか**で、裏へ渡すパスが変わる。
```nginx
# 末尾スラッシュ「あり」→ location 部分を置換して渡す
location /api/ { proxy_pass http://app/; }
#   /api/users  →  http://app/users      （/api を剥がす）

# 末尾スラッシュ「なし」→ パスをそのまま渡す
location /api/ { proxy_pass http://app; }
#   /api/users  →  http://app/api/users  （/api が残る）
```
「裏アプリが`/api`付きで404」「逆に剥がしすぎて404」はほぼこれ。**意図したパスになるか必ず確認**。

## root と alias / try_files
```nginx
# root: location名を「足して」探す
location /static/ { root /var/www; }   # /static/a.png → /var/www/static/a.png

# alias: location名を「置き換えて」探す
location /static/ { alias /var/www/assets/; }  # /static/a.png → /var/www/assets/a.png

# try_files: 順に探し、無ければ最後へフォールバック（SPA定番）
location / { try_files $uri $uri/ /index.html; }
```

## よく使う変数
| 変数 | 意味 |
|---|---|
| `$uri` | 正規化後のパス（クエリ無し） |
| `$request_uri` | 元のパス＋クエリ |
| `$host` / `$http_host` | ホスト名 |
| `$remote_addr` | 接続元IP（プロキシ下では前段のIP） |
| `$scheme` | http / https |
| `$args` / `$arg_xxx` | クエリ全体 / 個別パラメータ |
| `$upstream_response_time` | 裏アプリの応答時間（ログ・遅延切り分けに必須） |

## 基本の書き方（実用設定）
SPA配信＋API取り次ぎ＋HTTPS（定番フル例）:
```nginx
upstream app {
    server 127.0.0.1:3000;      # 複数書けば負荷分散
    keepalive 32;               # 接続再利用（要 proxy_http_version 1.1）
}

server {                        # HTTP は HTTPS へ
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    root /var/www/app/dist;
    gzip on;
    client_max_body_size 20m;   # アップロード上限

    location /api/ {
        proxy_pass http://app/;          # ← 末尾スラッシュ注意
        proxy_http_version 1.1;
        proxy_set_header Connection "";   # keepalive用
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }

    location / { try_files $uri $uri/ /index.html; }

    location ~* \.(jpg|png|css|js|woff2)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    location ~ /\.(?!well-known) { deny all; }   # .git/.env 等を遮断
}
```
運用コマンド:
```bash
nginx -t                       # 構文チェック（本番反映前は必須）
nginx -T                       # 全設定を展開表示（include含む）
sudo systemctl reload nginx    # 無停止リロード（restartではなくreload）
nginx -s reload                # 同上（master経由）
```

## 実務での使い方・定番パターン
- **SPA + API**: 静的を `root`、`/api/` だけ `proxy_pass`、それ以外は `try_files … /index.html`。
- **WebSocket**: `proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection "upgrade";` を追加。
- **複数サービス集約**: `server_name` ごとにバーチャルホスト、パスで `upstream` を分ける。
- **Dockerで前段**: `nginx:alpine` を置き、`upstream` をサービス名で指す（Compose/ECS）。→ [../docker/compose.md](../docker/compose.md)
- **PHPは PHP-FPM へ**: `location ~ \.php$ { fastcgi_pass 127.0.0.1:9000; include fastcgi_params; }`。→ [アプリサーバ.md](./アプリサーバ.md)
- **マイクロキャッシュ**: `proxy_cache` で1秒キャッシュし、スパイクを吸収。→ [チューニング.md](./チューニング.md)
- **本物IPの復元**: 前段にALB/CDNがある場合は `set_real_ip_from` / `real_ip_header` でクライアントIPを復元。→ [セキュリティ.md](./セキュリティ.md)

## ハマりどころ / アンチパターン
- **`reload` 忘れ**: 設定変更が反映されない。`nginx -t` → `reload` をセットで。
- **`proxy_pass` の末尾スラッシュ**: 付け外しでパスが変わり404。上記表を確認。
- **`location` 優先順位の誤解**: `^~` 忘れで静的が正規表現側に吸われる／`=`で固定すべきものを前方一致にする。
- **`502 Bad Gateway`**: 裏アプリ停止/ポート違い/ソケットパス違い。`upstream` と実待受を確認。→ [アプリサーバ.md](./アプリサーバ.md)
- **`413 Request Entity Too Large`**: `client_max_body_size` を上げる。
- **`504 Gateway Timeout`**: 重いAPIで `proxy_read_timeout` 超過。値調整＋アプリ側も見直す。
- **`try_files` 抜け**: SPAでリロード404。`/index.html` フォールバックを置く。
- **`add_header` が消える**: `location` で別の`add_header`を書くと親の分が継承されない。各所で再宣言 or `always`。→ [セキュリティ.md](./セキュリティ.md)
- **keepalive効かず接続を使い捨て**: upstream `keepalive` には `proxy_http_version 1.1; proxy_set_header Connection "";` が必須。
- **WebSocketが切れる**: `Upgrade`/`Connection` ヘッダ未設定。

## 関連
[仕組み.md](./仕組み.md) / [apache.md](./apache.md) / [リバースプロキシ.md](./リバースプロキシ.md) / [アプリサーバ.md](./アプリサーバ.md) / [チューニング.md](./チューニング.md) / [セキュリティ.md](./セキュリティ.md) / [ログ.md](./ログ.md)
配信・CDN: [../aws/cloudfront_route53.md](../aws/cloudfront_route53.md) ／ コンテナ化: [../docker/](../docker/) ／ 性能限界: [../負荷検証/指標.md](../負荷検証/指標.md)
