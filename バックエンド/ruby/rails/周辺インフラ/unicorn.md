# Unicorn（Rails の関連事項 / 周辺インフラ）

## ひとことで言うと
**プロセスフォーク型（マルチプロセス）の Rack アプリケーションサーバ**。マスタープロセスが複数の worker プロセスを fork し、各 worker が1リクエストずつ処理する。枯れていて安定。

## 役割・なぜ必要か
- RailsアプリをHTTPで動かす「アプリケーションサーバ」。ブラウザ↔（nginx）↔Unicorn↔Rails の最後の実行役。
- **1 worker = 1プロセスで1リクエスト**（プロセス内はシングルスレッド前提）。プロセスが完全分離されるため、1つが暴走/クラッシュしても他workerへの影響が小さい＝堅牢。

## 基本の書き方（設定・起動）
```ruby
# config/unicorn.rb
worker_processes ENV.fetch("WEB_CONCURRENCY", 3).to_i
preload_app true                       # アプリを先読み（Copy-on-Writeでメモリ節約）
listen ENV.fetch("PORT", 8080)
timeout 30

before_fork do |server, worker|
  ActiveRecord::Base.connection.disconnect! if defined?(ActiveRecord::Base)
end
after_fork do |server, worker|
  ActiveRecord::Base.establish_connection if defined?(ActiveRecord::Base)  # fork後にDB再接続
end
```
```bash
bundle exec unicorn -c config/unicorn.rb -E production -D
```

## 実務での使い方・定番パターン
- **前段に nginx 必須**：Unicorn は「遅いクライアント」に弱い。nginx にリクエスト/レスポンスのバッファリングを任せ、Unicorn は速いローカル通信だけ捌く構成にする。
- **worker 数**：CPUコア数を目安に、メモリと相談して決める（1 worker ≒ 1プロセス分のメモリ）。
- **メモリ肥大対策**：`unicorn-worker-killer` で肥大した worker を定期再起動。
- **ゼロダウンタイム再起動**：マスタへ `USR2` シグナル → 新マスタ起動 → 旧を落とす。
- `preload_app true` ＋ `before_fork`/`after_fork` でDB/Redis接続を正しく張り直す。

## ハマりどころ / アンチパターン
- **nginx 無しで公開**：遅いクライアントが worker を占有してスループット激減。
- **fork 後の接続引き継ぎ**：DB/Redis コネクションを `after_fork` で再接続しないと不整合・エラー。
- **メモリ食い**：マルチプロセスゆえ worker 数 × アプリメモリ。スレッド型より重い。
- **スレッド非対応コード前提**：プロセス分離なので並行性はプロセス数まで（I/O待ちが多いとPumaの方が効率的なことが多い）。

## Puma との使い分け
| | Unicorn | Puma |
|---|---|---|
| モデル | マルチプロセスのみ | マルチスレッド（＋クラスタでマルチプロセス） |
| メモリ | 重め（プロセス数分） | 効率的 |
| I/O並行 | 弱い | 強い |
| Rails既定 | × | ○（現行の既定） |
| 立ち位置 | 枯れて安定・レガシー/特定運用 | 標準 |

- 現在のRails既定は **Puma**。Unicorn は安定志向や既存資産で使われる。後継として **Pitchfork**（37signals、CoW最適化のfork型）もある。

## 補足: 「I/O並行」とは（Unicornが弱い理由）
**I/O（DB・外部API・ファイル・ネットワーク）の“待ち時間”の間に、別のリクエストを処理して待ちを重ね合わせること**。

Webアプリの時間の多くは「待ち」が占める:
```
1リクエスト: [CPUで少し計算] → [DBクエリ待ち .....] → [外部API待ち ....] → [少し計算]
                                  ↑この間 CPU は暇
```
この待ちの間に別のリクエストを進められると、同じリソースで多く捌ける（＝I/O並行が強い）。

### Rubyの事情（GVL）
Rubyには **GVL（Global VM Lock）** があり「**1プロセス内で同時にRubyコードを実行できるのは1スレッドだけ**」。

| | 説明 | Rubyでの扱い |
|---|---|---|
| **CPU並列** | 複数コアで計算そのものを同時実行 | GVLで不可 → **プロセスを増やす**（Unicorn / Pumaクラスタ） |
| **I/O並行** | 待ち時間を重ねて同時に“待つ” | **可能**。GVLは**I/O待ちの間は解放**され、別スレッドが動ける |

### Unicornだとどうなるか
Unicornは **1 worker（1プロセス）＝1リクエスト・シングルスレッド**。よって DBや外部API を待っている間、その worker は**まるごと暇**になり他のリクエストを処理できない（＝I/O並行が弱い）。同じ同時処理数を出すには worker（＝プロセス＝メモリ）を増やすしかない。
一方 Puma はマルチスレッドなので、1プロセス内でスレッドAがDB待ちの間にスレッドBが動ける＝待ち時間を埋められる。多くのWebアプリはI/Oバウンド（待ちが多い）なので、ここがPumaの効きどころ。

> 補足の補足: 「計算が重い（CPUバウンド）」処理はスレッドでは速くならない（GVLのため）。それはプロセスを増やすか、処理自体をジョブに逃がす（→ [sidekiq.md](./sidekiq.md) / [solid_queue.md](./solid_queue.md)）。

## 関連
[redis.md](./redis.md)（← puma.md / nginx.md は今後追加）
