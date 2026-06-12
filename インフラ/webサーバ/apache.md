# Apache HTTP Server（Webサーバ）

## ひとことで言うと
**Webサーバの歴史的定番**。`.htaccess` による**ディレクトリ単位の柔軟な設定**と、豊富なモジュール（mod_php / mod_rewrite 等）が特徴。LAMP（Linux+Apache+MySQL+PHP）の "A"。新規は nginx が増えたが、既存資産・共有レンタルサーバ・WordPress系で今も現役。処理モデルは **MPM** で選べる。

## 役割・なぜ必要か
- 静的配信・リバースプロキシ・TLS終端は nginx と同様に担える。
- 強みは **モジュール拡張** と **`.htaccess`**：アプリ側ディレクトリに設定ファイルを置くだけで本体設定を触らず挙動を変えられる（共有ホスティングで重宝）。
- 歴史が長く**情報・実績が膨大**。古いPHPアプリは Apache 前提が多い。
- 弱点は**高同時接続**：preforkは接続ごとにプロセスを使いメモリを食う（→ event MPM＋FPMで改善）。→ [仕組み.md](./仕組み.md)

## MPM（処理モデル）の選択
Apacheは「どう接続を捌くか」を **MPM（Multi-Processing Module）** で切り替える。
| MPM | 方式 | メモリ | 高同時接続 | mod_php | 推奨 |
|---|---|---|---|---|---|
| **prefork** | 接続ごとにプロセス | 多い | 弱い | 使える（スレッド非安全でも可） | 旧来のmod_php専用 |
| **worker** | プロセス×スレッド | 中 | 中 | 不可（要スレッド安全） | 中規模 |
| **event** | workerの改良（keep-alive専用スレッド） | 少 | **強い** | 不可 | **現代の既定** |
- **現在の推奨は `event` MPM ＋ PHP-FPM**。preforkでmod_phpは高負荷で破綻しやすい。
```bash
# 現在のMPM確認・切替（Debian系）
apachectl -V | grep MPM
sudo a2dismod mpm_prefork && sudo a2enmod mpm_event
```

## 設定の階層と適用範囲
```
 httpd.conf / apache2.conf          ← 全体
   └ conf-enabled/*.conf            ← 機能別（Debianは conf-available + a2enconf）
   └ sites-enabled/*.conf           ← サイト別（sites-available + a2ensite）
        └ <VirtualHost>             ← 1サイト（ホスト/ポート単位）
             └ <Directory>          ← ファイルシステムのパス単位
             └ <Location>           ← URL単位
             └ <Files>              ← ファイル名単位
                  └ .htaccess       ← 各ディレクトリに置く分散設定
```
- 適用は「広い→狭い」で上書き。`<Directory>` はパス、`<Location>` はURLに効く。
- Debian系は `a2ensite`/`a2enmod`/`a2enconf` で `*-available` → `*-enabled` へシンボリックリンクを張って有効化する。

## .htaccess（分散設定）
ディレクトリに置くだけで本体設定なしに挙動を変えられる。
```apache
# 例: HTTPS強制 + 末尾index + 隠しファイル遮断
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
```
- **有効化には本体側で `AllowOverride` が必要**：
```apache
<Directory /var/www/app>
    AllowOverride All     # None=.htaccess無視 / All=全許可 / FileInfo等=個別
</Directory>
```
- **性能コスト**：`.htaccess` は**リクエストごとに毎回**ディレクトリを遡って読む。本番で性能重視なら本体の `<Directory>` に書き、`AllowOverride None` にするのが定石。
- 使いどころ：本体設定を触れない**共有レンタルサーバ**、アプリ同梱のリライト（WordPress等）。

## mod_rewrite（URL書き換え）
```apache
RewriteEngine On

# WWWなし→あり に統一
RewriteCond %{HTTP_HOST} ^example\.com$ [NC]
RewriteRule ^ https://www.example.com%{REQUEST_URI} [L,R=301]

# SPAフォールバック（実ファイルが無ければ index.html）
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^ index.html [L]

# メンテナンス画面（特定IP以外を503へ）
RewriteCond %{REMOTE_ADDR} !^203\.0\.113\.10$
RewriteRule ^ /maintenance.html [R=503,L]
```
主要フラグ：
| フラグ | 意味 |
|---|---|
| `[L]` | これで書き換え終了（last） |
| `[R=301]` / `[R=302]` | リダイレクト（恒久 / 一時） |
| `[QSA]` | 元のクエリ文字列を引き継ぐ |
| `[NC]` | 大小無視 |
| `[P]` | プロキシ（mod_proxy へ渡す） |
| `[F]` | 403を返す（forbidden） |

## リバースプロキシ（mod_proxy）
```apache
# 要: a2enmod proxy proxy_http
<VirtualHost *:443>
    ServerName example.com
    DocumentRoot /var/www/app/dist

    SSLEngine on
    SSLCertificateFile    /etc/letsencrypt/live/example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem

    ProxyPreserveHost On
    ProxyPass        /api/ http://127.0.0.1:3000/
    ProxyPassReverse /api/ http://127.0.0.1:3000/
    RequestHeader set X-Forwarded-Proto "https"
</VirtualHost>
```
ロードバランス（`mod_proxy_balancer`）:
```apache
<Proxy balancer://appcluster>
    BalancerMember http://10.0.0.11:3000
    BalancerMember http://10.0.0.12:3000
</Proxy>
ProxyPass        /api/ balancer://appcluster/
ProxyPassReverse /api/ balancer://appcluster/
```

## 主要モジュール
| モジュール | 用途 |
|---|---|
| `mod_ssl` | TLS/HTTPS |
| `mod_rewrite` | URL書き換え |
| `mod_proxy` / `mod_proxy_http` | リバースプロキシ |
| `mod_proxy_fcgi` | PHP-FPM連携（FastCGI） |
| `mod_headers` | レスポンスヘッダ付与（セキュリティヘッダ） |
| `mod_deflate` | gzip圧縮 |
| `mod_expires` | キャッシュ制御 |
| `mod_status` | 稼働状況の可視化 |
| `mod_security` | WAF |

## mod_php から PHP-FPM へ
```apache
# 旧: mod_php（Apacheプロセス内でPHP実行。preforkに縛られる）
# 新: PHP-FPM + mod_proxy_fcgi（推奨）
<FilesMatch \.php$>
    SetHandler "proxy:unix:/run/php/php8.3-fpm.sock|fcgi://localhost/"
</FilesMatch>
```
- mod_phpは「Apacheと同居して簡単」だが prefork 必須でスケールしにくい。
- **PHP-FPM分離**なら event MPM が使え、PHPのプロセス数を独立にチューニングできる。→ [アプリサーバ.md](./アプリサーバ.md)

## 運用・ログ
```bash
apachectl configtest        # = nginx -t（構文チェック）
sudo systemctl reload apache2   # graceful（無停止リロード）
apachectl -M                # 有効モジュール一覧
```
```apache
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
CustomLog /var/log/apache2/access.log combined
ErrorLog  /var/log/apache2/error.log
```

## 実務での使い方・定番パターン
- **共有レンタルサーバ / 既存PHP / WordPress**: `.htaccess` でリライト・Basic認証・キャッシュ。
- **event MPM ＋ PHP-FPM** を基本に（prefork+mod_phpは避ける）。
- **既存Apache案件の保守**が中心用途。新規で前段だけ欲しいなら nginx を選ぶことが多い。→ [比較.md](./比較.md)
- TLSは Let's Encrypt（certbot の Apacheプラグイン）で自動更新。

## ハマりどころ / アンチパターン
- **`.htaccess` が効かない**: `AllowOverride None`。`<Directory>` で `AllowOverride All`（or 必要ディレクティブ）に。
- **`.htaccess` 多用で遅い**: 毎リクエスト読み込む。性能重視は本体設定へ移し `AllowOverride None`。
- **モジュール有効化忘れ**: `ProxyPass`に`mod_proxy`/`mod_proxy_http`、リライトに`mod_rewrite`（`a2enmod`）。
- **prefork + 高同時接続でメモリ枯渇**: event MPM＋FPMへ。
- **`ProxyPassReverse` 忘れ**: リダイレクト時の Location ヘッダに内部アドレスが漏れる。
- **反映漏れ**: `configtest` → `reload`（graceful）を徹底。restartは接続を切る。
- **mod_phpのままスケールしようとする**: preforkに縛られ頭打ち。FPM分離が前提。

## 関連
[仕組み.md](./仕組み.md) / [nginx.md](./nginx.md) / [リバースプロキシ.md](./リバースプロキシ.md) / [アプリサーバ.md](./アプリサーバ.md) / [比較.md](./比較.md)
PHP実行: [../../バックエンド/php/](../../バックエンド/php/) ／ コンテナ化: [../docker/](../docker/) ／ ログ運用: [ログ.md](./ログ.md)
