# Ruby 3.4（言語解説）

## ひとことで言うと
すべてが式でありオブジェクトである動的型付け言語。`each` / `map` のようなブロック付きメソッドとミックスイン（module）で組み立て、メタプログラミングで DSL まで作れる。3.4 では YJIT が標準で安定し、`it` ブロック引数と新パーサ Prism が入った。

## このバージョンの位置づけ（リリース時期 / サポート・EOL / どこで使うか）
- リリース: Ruby 3.4.0 は 2024年12月25日（恒例のクリスマスリリース）。
- サポート: Ruby は各マイナー版を約3年メンテする。3.4 系は通常ライン（バグ修正）→セキュリティ修正ラインを経て、目安として2027年末ごろEOL。本番では「サポート中の最新2〜3マイナー」を使うのが安全。
- どこで使うか: Web アプリ（Rails/Sinatra/Hanami）、CLI ツール、運用自動化スクリプト、RSpec/Rake のような DSL。`ruby -v` でバージョン、`bundle exec` で依存解決済みの環境で実行する。

## 言語の基本（文法の要点）
- すべてが式: `if` も `case` も値を返す。代入の右辺に直接書ける。
```ruby
status = if score >= 80 then "pass" else "fail" end
label  = case status
         in "pass" then "合格"
         in "fail" then "不合格"
         end
```
- メソッド定義と暗黙の戻り値: 最後に評価した式が戻り値。`return` は早期脱出のときだけ書く。
```ruby
def full_name(first, last)
  "#{first} #{last}"   # この行の値が戻り値
end
```
- キーワード引数（位置引数と区別される / 3.0 以降は明確に分離）:
```ruby
def create_user(name:, age: 20)
  { name: name, age: age }
end
create_user(name: "Aki", age: 30)
```
- ブロックは「メソッドに渡す無名の処理のかたまり」。`do...end` か `{ }`。
```ruby
[1, 2, 3].map { |n| n * 2 }       # => [2, 4, 6]
[1, 2, 3].each do |n|
  puts n
end
```
- パターンマッチ（`case/in`）で構造を分解:
```ruby
case { name: "Aki", role: :admin }
in { name: String => name, role: :admin }
  puts "#{name} は管理者"
in { name: String => name }
  puts "#{name} は一般"
end
```
- 安全ナビゲーション `&.`（nil なら呼ばずに nil を返す）:
```ruby
user&.profile&.bio   # user が nil でも例外にならない
```

## この言語の核心概念（他言語と違う・必ず押さえる）

### すべてが式・すべてがオブジェクト
- 何か: 文ではなく式で、`if`/`case`/メソッド定義まで値を返す。整数・`nil`・`true` もすべてオブジェクトでメソッドを持つ。
```ruby
x = if 1 > 0 then "yes" else "no" end   # if が値を返す
3.times { |i| puts i }                  # 整数 3 にも times メソッド
nil.to_s                                # => ""（nil もオブジェクト）
(-5).abs                                # => 5
"abc".class.ancestors                   # => [String, Comparable, Object, ...]
```
- 違う点 / つまずき: C/Java/Python では `if` は文で値を返さないが Ruby は返す。`5.foo` のように数値にもメソッドが呼べる。「プリミティブ型」という特別枠が無い（`Integer` も普通のクラス）。

### 真偽値は `nil` と `false` だけが偽
- 何か: 条件判定で偽になるのは `nil` と `false` の2つだけ。それ以外（`0`・`""`・`[]`）はすべて真。
```ruby
puts "truthy" if 0      # => truthy（0 も真）
puts "truthy" if ""     # => truthy（空文字も真）
puts "truthy" if []     # => truthy（空配列も真）
puts "falsy"  unless nil  # => falsy
```
- 違う点 / つまずき: JavaScript/Python では `0`・`""`・空配列が偽になるが、Ruby では真。`if count == 0` のつもりで `if count` と書くと 0 でも真になりバグる。空判定は `.empty?` / `.zero?` を明示する。

### ブロック / yield / Proc / lambda
- 何か: ブロックは「メソッドに渡す無名の処理」。`{ }`（1行）か `do...end`（複数行）。メソッド側は `yield` で実行、`&block` で受け取って持ち回せる。Proc と lambda は引数チェックと `return` の挙動が違う。
```ruby
def each_twice(arr)
  arr.each { |x| yield x; yield x }   # yield で渡されたブロックを実行
end
each_twice([1, 2]) { |n| print n }    # => 1122

def wrap(&block)                       # &block でブロックを Proc として受け取る
  "[#{block.call}]"
end
wrap { "hi" }                          # => "[hi]"

prc = Proc.new { return 10 }           # Proc の return は呼び出し元メソッドごと脱出
lam = lambda  { return 10 }            # lambda の return は lambda 自身だけ脱出
add = ->(a, b) { a + b }               # ->() は lambda の短縮記法
add.call(1, 2)                         # => 3
[1, 2, 3].map(&:to_s)                  # &:sym でメソッドをブロック化 => ["1","2","3"]
```
- 違う点 / つまずき: lambda は引数の数を厳密にチェックし `return` で自分だけ抜けるが、Proc は引数がゆるく `return` するとメソッド全体を抜ける（ループ内で Proc を呼ぶと意図せず関数を抜けて事故る）。多くの言語の「関数オブジェクト」は lambda 寄りの挙動だと考えると整理しやすい。

### Mixin（module の include / extend / prepend）
- 何か: Ruby は単一継承だが、module を混ぜ込んで振る舞いを合成する（多重継承の代替）。`include`＝インスタンスメソッドに追加、`extend`＝特異（クラス）メソッドに追加、`prepend`＝既存メソッドの前に差し込む。
```ruby
module Greetable
  def greet = "Hi, #{name}"
end
module Loud
  def greet = super.upcase     # prepend すると元の greet を super で呼べる
end

class User
  include Greetable            # インスタンスメソッドとして greet を獲得
  prepend Loud                 # greet 呼び出しが Loud → Greetable の順になる
  attr_reader :name
  def initialize(name) = @name = name
end
User.new("Aki").greet          # => "HI, AKI"
User.ancestors                 # => [Loud, User, Greetable, Object, ...]
```
- 違う点 / つまずき: Java の interface と違い module は実装を持てる。メソッド解決順序（`ancestors`）を理解しないと、どの定義が勝つか分からなくなる。`include`（self に追加）と `extend`（特異メソッドに追加）を取り違えるのが定番ミス。

### シンボル vs 文字列
- 何か: `:name` のシンボルは「同じ綴りなら同一オブジェクト」になる不変の名札。同じ文字列リテラルは毎回別オブジェクト。
```ruby
:name.object_id == :name.object_id     # => true（常に同一）
"name".object_id == "name".object_id   # => false（毎回新規生成）
{ name: "Aki" }                        # ハッシュキーはシンボルが定番
:active.to_s                           # => "active"
"active".to_sym                        # => :active
```
- 違う点 / つまずき: ハッシュのキーをシンボルで作ったか文字列で作ったかで取り出せず詰まる（`h[:k]` と `h["k"]` は別物）。外部入力（JSON のキーなど）は文字列で来るので `symbolize_keys` 等で変換が要る。他言語の「インターン済み文字列」に近いが、文字列とは別の型である点に注意。

### 特異メソッド / オープンクラス（後からメソッド追加）
- 何か: 既存のクラスやインスタンスに後からメソッドを足せる。特定インスタンスだけに付くのが特異メソッド、クラス全体を開いて足すのがオープンクラス。
```ruby
class String
  def shout = "#{upcase}!"     # 既存の String に後からメソッド追加
end
"hi".shout                     # => "HI!"

obj = Object.new
def obj.special = "only me"    # この obj インスタンスだけに付く特異メソッド
obj.special                    # => "only me"
```
- 違う点 / つまずき: 標準クラスにメソッドを足せる強力さの裏返しで、gem 同士が同じメソッドを定義して衝突する「モンキーパッチ汚染」が起きる。Rails の `5.days.ago` 等はこの仕組み。多用すると挙動の出どころが追えなくなるため `refine`（局所適用）も検討する。

### メタプログラミング（define_method / method_missing / send）
- 何か: 実行時にメソッドを定義したり、未定義呼び出しを捕まえたり、名前を文字列で動的に呼べる。RSpec / Rails の DSL の土台。
```ruby
class Config
  [:host, :port].each do |key|
    define_method(key) { @data&.dig(key) }            # 実行時にメソッド生成
  end
  def method_missing(name, *args)                     # 未定義メソッドを捕捉
    name.to_s.end_with?("=") ?
      (@data ||= {})[name.to_s.chomp("=").to_sym] = args.first : super
  end
  def respond_to_missing?(*) = true
end
c = Config.new
c.host = "localhost"
c.send(:host)                  # => "localhost"（send で名前を動的に呼ぶ）
```
- 違う点 / つまずき: `method_missing` を定義したら必ず `respond_to_missing?` も合わせる（さもないと `respond_to?` が嘘をつく）。`send` は private メソッドも呼べてしまうので外部入力を直接渡さない。便利だが grep で定義箇所が見つからず読みにくくなる諸刃の剣。

### 引数の渡し方（参照の値渡し）
- 何か: Ruby は「参照の値渡し」。引数の指すオブジェクトの**中身を変えれば呼び出し元に伝わる**が、引数変数を**別オブジェクトに再代入しても伝わらない**。Array/Hash も参照なので `push` 等は伝わる。Ruby は**文字列が可変**なので `s << "x"` も伝わる点が Java/Python と違う。
- 詳しい比較（Java/Python/PHP/Go/Swift との違い・メモリ図）→ [../../../言語共通概念/値渡しと参照.md](../../../言語共通概念/値渡しと参照.md)

## 型・データモデル
- 動的型付け＋ダックタイピング: 「クラスが何か」ではなく「そのメソッドに応答するか」で扱う。`respond_to?(:each)` が真なら each できる。
- すべてがオブジェクト: 整数も `nil` も `true` もオブジェクトでメソッドを持つ。
```ruby
3.times { |i| puts i }     # 整数 3 にも times メソッド
nil.to_s                   # => ""（nil もメソッドを持つ）
(-5).abs                   # => 5
```
- 主なリテラル: 数値（`Integer`/`Float`/`Rational` `3r`/`Complex` `2i`）、`String`、`Symbol`（`:name` 不変の名札）、`Array`、`Hash`、`Range`（`1..10` / 終端省略 `1..`）。
- ミュータブル/イミュータブル: 文字列は可変。`freeze` で凍結。Symbol は不変なのでハッシュキーに向く。
- 型注釈は言語コア外: 別ファイル `.rbs`（RBS）に書き、`steep` や Sorbet で静的検査する。コードに直接書かないのが Ruby 流。
```ruby
# sig/user.rbs
# class User
#   def name: () -> String
# end
```

## この言語らしさ / 特徴的な機能
（中核となる Mixin・ブロック/Proc/lambda・メタプログラミング・特異メソッドは「この言語の核心概念」を参照）
- イテレータと `Enumerable`: `map`/`select`/`reduce`/`each_with_object` など。自作クラスも `each` を定義して `include Enumerable` すれば全部使える。
```ruby
[1, 2, 3, 4].select(&:even?).sum    # => 6
```
- 例外処理は `begin/rescue/ensure`。`raise` で送出、`retry` で再試行。

## 並行・非同期
- スレッドと GVL（旧称 GIL）: 1つの Ruby プロセス内で同時に走る Ruby コードは基本1スレッド。I/O 待ち（DB・HTTP・ファイル）の間は他スレッドが動けるので **I/O 並行は有効**、CPU バウンドな並列計算は苦手。
- Fiber: 軽量な協調的並行。自分で `Fiber.yield` / `resume` して切り替える。Async gem や 3.x の Fiber スケジューラと組むとノンブロッキング I/O を素直に書ける。
```ruby
fib = Fiber.new do
  Fiber.yield 1
  Fiber.yield 2
  3
end
fib.resume   # => 1
fib.resume   # => 2
```
- Ractor: GVL を回避して**真の並列**を狙う仕組み。Ractor 間はオブジェクトを共有せずメッセージで受け渡す（共有可能なのは不変オブジェクトのみ）。3.4 時点でも experimental 扱いで、使用すると警告が出る。
```ruby
r = Ractor.new { Ractor.receive * 2 }
r.send(21)
r.take       # => 42
```
- 重い並列処理は外部に逃がすのが定番（プロセス分割、Sidekiq などのジョブキュー）。

## 標準ライブラリ / ツールチェーン
- パッケージ管理: `gem`（個々のライブラリ）と `bundler`（プロジェクトの依存をまとめて固定）。`Gemfile` に書いて `bundle install`、`Gemfile.lock` でバージョンを固定する。
- バージョン管理: `rbenv` / `rvm` / `mise`（README 参照）。
- 標準ライブラリ例: `json`、`net/http`、`set`、`securerandom`、`logger`、`fileutils`、`open3`（外部コマンド実行）、`date`/`time`。
- よく使う組み込み: `Enumerable`、`Comparable`、`Struct`（軽い値オブジェクト）、`Data`（3.2+ の不変値オブジェクト）。
```ruby
Point = Data.define(:x, :y)
p = Point.new(x: 1, y: 2)
p2 = p.with(y: 9)        # 新インスタンスを返す（不変）
```
- 品質ツール: `RuboCop`（lint/format）、テストは `Minitest`（標準同梱）/ `RSpec`。

## このバージョンの新機能・トピック
- `it` ブロック引数: 引数1つのブロックで `|x|` を書かずに `it` で参照できる（`_1` の読みやすい別名）。
```ruby
[1, 2, 3].map { it * 2 }    # => [2, 4, 6]（|n| n*2 と同じ）
```
- 新パーサ Prism がデフォルトに: Ruby 本体のパーサが Prism に置き換わり、構文エラーメッセージが分かりやすくなり、エディタ/ツール（Rubocop, RBS, ruby-lsp 等）が同じパース結果を共有しやすくなった。
- YJIT のさらなる成熟: JIT コンパイラがメモリ効率・速度ともに改善し、Rails 等の実アプリで標準的に有効化して使える段階に。`--yjit` または環境変数で有効化。
- 文字列リテラルの凍結に向けた移行: `# frozen_string_literal: true` マジックコメントがない場合の挙動について将来の凍結デフォルト化を見据えた警告/方針が進行中（チャンクごとに移行が進む）。新規ファイルでは付ける癖をつけると安全。
- `Hash#freeze` 周りや `Range#step` などの細かな改善、`require` 性能改善なども入る。

## ハマりどころ
- ブロック引数の `it`: 3.4 で `it` がブロック引数になったため、メソッド名 `it`（RSpec の `it` など）と衝突しない文脈かを意識する。明示メソッド呼び出し `it(...)` やレシーバ付きなら従来通り。
- `nil` とぼっち演算子: `obj.method` が落ちるのは `obj` が nil のとき。連鎖は `&.` で守るが、付けすぎるとバグ（nil が静かに伝播）を見逃す。
- ミュータブルな文字列の共有: 同じ文字列オブジェクトを複数所有して片方を `<<` で破壊的更新すると両方変わる。`dup` / `frozen_string_literal` で防ぐ。
- キーワード引数 vs ハッシュ: 3.0 以降は位置引数の最後のハッシュが自動でキーワードに化けない。`**` で明示的に展開する。
- GVL の誤解: マルチスレッドにしても CPU 計算は速くならない。並列したいなら Ractor / プロセス / ネイティブ拡張へ。
- Ractor の実用度: experimental かつ「共有不可」制約が厳しく、既存 gem がそのまま動かないことが多い。本番投入は慎重に。

## 関連
- フレームワーク: [../rails/](../rails/)（Ruby on Rails）/ [../sinatra/](../sinatra/)（軽量）/ [../hanami/](../hanami/)（モジュラー）。
- 言語概要: [../README.md](../README.md)。
