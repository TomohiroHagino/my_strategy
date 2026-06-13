# Python 3.13（言語解説）

## ひとことで言うと
インデントで構造を表す汎用の動的言語。内包表記・デコレータ・ダックタイピング＋型ヒントで書き、`asyncio` で非同期 I/O を回す。3.13 では experimental な free-threaded ビルド（GIL を外せる）と JIT、対話シェル（REPL）の刷新が目玉。

## このバージョンの位置づけ（リリース時期 / サポート・EOL / どこで使うか）
- リリース: Python 3.13.0 は 2024年10月7日。
- サポート: 各マイナー版は約5年（バグ修正2年＋セキュリティ修正3年）。3.13 のEOLは2029年10月ごろ。本番はサポート中の最新2〜3マイナー（3.11〜3.13）が無難。
- どこで使うか: Web/API（Django・FastAPI・Flask）、データ分析（pandas・Polars・NumPy）、AI/ML（PyTorch・scikit-learn）、自動化・CLI。環境隔離と実行は [../環境管理.md](../環境管理.md) を参照（venv / uv / Docker の棲み分け）。

## 言語の基本（文法の要点）
- インデントがブロックを表す（波括弧なし）。コロン `:` の後に字下げ。
```python
def classify(score: int) -> str:
    if score >= 80:
        return "pass"
    else:
        return "fail"
```
- 内包表記（リスト/辞書/集合/ジェネレータ）: ループを1行で書く Python 的記法。
```python
squares = [n * n for n in range(5)]          # [0, 1, 4, 9, 16]
evens   = {n for n in range(10) if n % 2 == 0}
by_name = {u.id: u.name for u in users}
```
- デコレータ: 関数/メソッドを別の関数で包んで機能を足す `@` 構文。
```python
import functools

def logged(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        print(f"call {fn.__name__}")
        return fn(*args, **kwargs)
    return wrapper

@logged
def add(a, b):
    return a + b
```
- f-string（文字列補間）と複数代入:
```python
name, age = "Aki", 30
msg = f"{name} is {age}"
```
- 構造的パターンマッチ（`match/case`・3.10+）:
```python
match command:
    case {"type": "move", "x": int(x), "y": int(y)}:
        move(x, y)
    case ["quit"]:
        stop()
    case _:
        unknown()
```
- 例外処理 `try/except/else/finally`、例外グループ `except*`（3.11+）。

## この言語の核心概念（他言語と違う・必ず押さえる）

### すべてオブジェクト＆名前は参照（is vs ==・可変デフォルト引数の罠）
- 何か: Python の変数は「箱」ではなく値への名札（参照）。代入は同じオブジェクトに別名を付けるだけ。`is` は同一オブジェクト判定、`==` は値の等価判定。
```python
a = [1, 2, 3]
b = a                 # 同じリストに別名を付けただけ
b.append(4)           # a も [1,2,3,4] に変わる（共有している）
c = a.copy()          # 独立した別オブジェクト
a is b                # True（同一オブジェクト）
a is c                # False / a == c  # True（値は等しい）

def append_to(item, target=[]):   # ★デフォルト引数は定義時に1度だけ生成され使い回される
    target.append(item)
    return target
append_to(1)          # [1]
append_to(2)          # [1, 2] ← 前回のリストが残る！ 事故の定番
```
- 違う点 / つまずき: 可変オブジェクト（list/dict/set）を代入・引数渡しすると共有される。デフォルト引数に `[]`/`{}` を置くと呼び出し間で使い回されてバグる→ `None` を既定にし関数内で生成する。`is` は `None` 判定（`x is None`）だけに使い、値比較は `==`。小整数や短い文字列は内部キャッシュで `is` がたまたま真になるので当てにしない。
- 関数引数の伝わり方（中身変更は伝わる／再代入は伝わらない）を Java/Ruby/PHP/Go 等と比べた図解 → [../../../言語共通概念/値渡しと参照.md](../../../言語共通概念/値渡しと参照.md)

### 内包表記（list / dict / set / generator）
- 何か: ループ＋条件＋変換を1行で書く Python の中核記法。角括弧で list、波括弧で set/dict、丸括弧で遅延評価のジェネレータ。
```python
squares = [n * n for n in range(5)]                  # [0,1,4,9,16]
evens   = {n for n in range(10) if n % 2 == 0}       # set
by_id   = {u.id: u.name for u in users}              # dict
gen     = (n * n for n in range(10**9))              # generator（メモリに展開しない）
matrix  = [[r * c for c in range(3)] for r in range(3)]   # ネスト
total   = sum(x for x in data if x > 0)              # 関数引数に直接渡せる
```
- 違う点 / つまずき: 多くの言語は `map`/`filter` のメソッドチェーンだが Python は内包表記が第一選択。`(... )` は tuple ではなくジェネレータ（1回しか回せず、`len()` も効かない）。ネストしすぎると一気に読めなくなるので、深い変換は通常のループに戻すのが Pythonic。

### デコレータ（関数を包む `@`）
- 何か: 関数/メソッドを別の関数で包んで前後処理や機能を足す構文。`@deco` は `f = deco(f)` の糖衣。
```python
import functools

def logged(fn):
    @functools.wraps(fn)                  # 元関数の名前/docstring を保つ
    def wrapper(*args, **kwargs):
        print(f"call {fn.__name__}")
        return fn(*args, **kwargs)
    return wrapper

@logged
def add(a, b):
    return a + b
add(1, 2)                                 # "call add" のあと 3

@functools.lru_cache(maxsize=None)        # 引数付きデコレータ＝デコレータを返す関数
def fib(n): return n if n < 2 else fib(n-1) + fib(n-2)
```
- 違う点 / つまずき: `@property` `@staticmethod` `@dataclass` など標準機能の多くがデコレータ。`@functools.wraps` を付けないと包んだ関数の `__name__` が `wrapper` になりデバッグやイントロスペクションが壊れる。引数付きデコレータは「デコレータを返す関数」になる入れ子構造で混乱しやすい。

### ダックタイピング ＋ 型ヒント（実行時は無視）
- 何か: 「型が何か」ではなく「そのメソッド/振る舞いがあるか」で動く。型ヒントは注釈に過ぎず実行時には強制されない（`mypy`/`pyright` が静的に検査するだけ）。
```python
def total(items: list[int]) -> int:       # 型ヒントは強制されない
    return sum(items)
total(["a", "b"])                         # 実行時エラーにならず sum で初めて落ちる

class Reader:                              # Iterable を継承しなくても…
    def __iter__(self): yield from [1, 2, 3]
for x in Reader(): print(x)               # __iter__ があれば for で回せる（ダックタイピング）

from typing import Protocol               # 構造的部分型でダックタイピングを型で表現
class HasName(Protocol):
    name: str
```
- 違う点 / つまずき: Java/Go の名目的型付けと違い、明示的な継承や interface 実装は不要（`__iter__`/`__len__` 等の特殊メソッドがあれば対応扱い）。型ヒントを書いても実行時の値チェックは行われないので、入力検証は `pydantic` 等で別途やる。型ヒントは新記法 `list[int]` `dict[str,int]` `X | None`（`Optional` の代わり）が標準。

### イテレータ / ジェネレータ（yield）と遅延評価
- 何か: `yield` を持つ関数はジェネレータになり、呼ばれた時点では実行されず、要素を要求されるたびに途中から再開して1つずつ返す（遅延評価）。巨大データをメモリに載せず流せる。
```python
def count_up(n):
    i = 0
    while i < n:
        yield i           # ここで一旦止まり、次に要求されたら続きから再開
        i += 1

g = count_up(3)
next(g)                   # 0
next(g)                   # 1
for x in count_up(10**9): # 10億要素でもメモリは一定
    if x > 2: break

def read_lines(path):     # ファイルを1行ずつ流す（全部読み込まない）
    with open(path) as f:
        for line in f:
            yield line.rstrip()
```
- 違う点 / つまずき: ジェネレータは1回しか消費できない（使い切ると空。再利用は作り直し）。`len()` は効かず、全部欲しいときは `list(g)`。`range`・`map`・`filter`・`dict.items()` も遅延評価の「ビュー/イテレータ」で、その場では中身が無い点が他言語の即時リストと違う。

### *args / **kwargs とアンパック
- 何か: `*args` は可変長の位置引数をタプルで、`**kwargs` はキーワード引数を辞書で受ける。呼び出し側は `*`/`**` でシーケンス/辞書を展開して渡せる。
```python
def f(*args, **kwargs):
    print(args, kwargs)
f(1, 2, x=3)                        # (1, 2) {'x': 3}

nums = [1, 2, 3]
print(*nums)                        # print(1, 2, 3) に展開
opts = {"sep": "-", "end": "!\n"}
print("a", "b", **opts)             # キーワードとして展開 → a-b!

first, *rest = [1, 2, 3, 4]         # first=1, rest=[2,3,4]
a, b = b, a                         # タプルアンパックで入れ替え
merged = {**d1, **d2}               # 辞書のマージ
```
- 違う点 / つまずき: 同じ `*`/`**` が「定義側＝集める」「呼び出し側＝展開する」で逆の意味になるのが混乱の元。デコレータの `wrapper(*args, **kwargs)` で任意の関数をそのまま中継できるのはこの仕組み。

### GIL（CPUバウンドで効かない並列／3.13 の no-GIL）
- 何か: 通常の CPython には GIL（グローバルインタプリタロック）があり、1プロセス内で同時に走る Python バイトコードは1スレッドだけ。I/O 待ちの間は GIL を手放すので I/O 並行は効くが、CPU 計算は複数スレッドにしても速くならない。
```python
import threading

def heavy():                              # 純 Python の CPU 計算
    s = 0
    for i in range(10_000_000): s += i

# 2スレッドにしても GIL のせいで CPU 時間はほぼ半分にならない（並行≠並列）
ts = [threading.Thread(target=heavy) for _ in range(2)]
for t in ts: t.start()
for t in ts: t.join()
```
- 違う点 / つまずき: Java/Go のスレッドは真の CPU 並列になるが、Python の `threading` は CPU バウンドでは無力。CPU 並列が要るなら `multiprocessing`（プロセス分割）か NumPy 等のネイティブ拡張（C 側で GIL を解放）に逃がす。3.13 では experimental な free-threaded ビルド（`python3.13t`）で GIL を外せるようになりつつあるが、互換性・性能はまだ発展途上で本番は様子見。

## 型・データモデル
- 動的型＋漸進的型付け（ダックタイピング）: 実行時は「そのメソッドがあるか」で動く。型ヒントは任意の注釈で、`mypy` / `pyright` が静的検査する（実行時には強制されない）。
```python
def total(items: list[int]) -> int:
    return sum(items)
```
- 組み込み型: `int`（多倍長・桁あふれなし）、`float`、`str`、`bool`、`bytes`、`list`、`tuple`（不変）、`dict`（挿入順保持）、`set`、`None`。
- すべてがオブジェクト＆第一級関数: 関数もクラスもオブジェクトで変数に入れ引数に渡せる（詳細は「この言語の核心概念」を参照）。
- ミュータブル/イミュータブル: `list`/`dict`/`set` は可変、`tuple`/`str`/`frozenset` は不変。
- データ構造の定番: `dataclasses`（軽い値クラス）、`typing` の `Protocol`（構造的部分型＝ダックタイピングを型で表す）。
```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Point:
    x: int
    y: int

p = Point(1, 2)          # p.x, p.y / 不変
```
- 型ヒントの新記法: `list[int]` `dict[str, int]` `X | None`（Optional の代わり）が標準。

## この言語らしさ / 特徴的な機能
（内包表記・デコレータ・ジェネレータ/`yield`・ダックタイピング・`*args`/`**kwargs` は「この言語の核心概念」を参照）
- 「読みやすさ」最優先（The Zen of Python, `import this`）。誰が書いても似た見た目になる。
- マルチパラダイム: 手続き・OOP・関数型（`map`/`filter`/`functools`）を混ぜて書ける。
- コンテキストマネージャ `with`: 後始末（ファイルクローズ等）を確実に行う。
```python
with open("data.txt") as f:
    text = f.read()
# ここで自動的に close される
```

## 並行・非同期
- GIL の基本は「この言語の核心概念」を参照（1スレッドしか Python バイトコードを実行できず、I/O 並行は有効だが CPU 並列は苦手）。
- `asyncio`（イベントループによる協調的非同期）: `async def` / `await` でノンブロッキング I/O を1スレッドで多重化。Web サーバ・大量の HTTP/DB アクセスに向く。
```python
import asyncio

async def fetch(name: str) -> str:
    await asyncio.sleep(1)          # I/O 待ちを模擬
    return f"done: {name}"

async def main() -> None:
    results = await asyncio.gather(fetch("a"), fetch("b"))   # 並行に待つ
    print(results)

asyncio.run(main())
```
- スレッド（`threading`）: I/O 並行向き。CPU は速くならない。
- プロセス（`multiprocessing`）/ ネイティブ拡張（NumPy 等）: GIL を回避して CPU 並列を実現する従来手段。
- 3.13 では experimental な free-threaded ビルド（後述）で GIL なし＝CPU 並列スレッドが可能になりつつある。

## 標準ライブラリ / ツールチェーン
- パッケージ/環境管理: `pip`（インストール）＋ `venv`（隔離）、高速な `uv` や Poetry も普及。詳細は [../環境管理.md](../環境管理.md)。
- 「電池付属」の厚い標準ライブラリ: `pathlib`（パス操作）、`json`、`datetime`、`collections`（`defaultdict`/`Counter`）、`itertools`、`functools`、`subprocess`、`logging`、`http`、`sqlite3`、`re`（正規表現）。
```python
from collections import Counter
Counter("banana")            # {'a': 3, 'n': 2, 'b': 1}
```
- 品質ツール: `ruff`（lint/format・高速）、`black`（整形）、`mypy`/`pyright`（型）。テストは `pytest` / `unittest`。
- 実行: Web は Gunicorn / Uvicorn（ASGI）＋ nginx。

## このバージョンの新機能・トピック
- experimental free-threaded（no-GIL）ビルド: GIL を外したビルド（`python3.13t`）が試験的に提供され、複数スレッドで Python コードを真に並列実行できる。まだ実験段階で互換性・性能に注意（一部拡張は未対応）。通常ビルドは従来どおり GIL あり。
- experimental JIT コンパイラ: ビルドオプションで有効化できる軽量 JIT が入った。ホットなコードを機械語化して将来的な高速化の土台を作る段階（既定では無効、効果は限定的）。
- 改良された対話シェル（REPL）: 複数行編集、色付き表示、履歴、`help`/`exit` がそのまま効くなど対話作業が快適に。
- エラーメッセージの改善: トレースバックに色が付き、原因箇所がより分かりやすく示される。
- 型周りの強化: `typing.TypeIs`（より正確な型ナローイング）、型パラメータのデフォルト（`TypeVar` のデフォルト・PEP 696）、`warnings.deprecated` デコレータで非推奨を型チェッカに伝えられる。
```python
from typing import TypeIs

def is_str_list(v: list[object]) -> TypeIs[list[str]]:
    return all(isinstance(x, str) for x in v)
```
- 標準ライブラリの整理: 古い「デッドバッテリー」モジュール群が削除（PEP 594）。`dbm.sqlite3` 追加など実用的な更新もあり。

## ハマりどころ
- 可変デフォルト引数: `def f(x=[])` は呼び出し間で同じリストを共有してしまう。`None` を既定にして関数内で生成する。
- インデント混在: タブとスペースの混在は構文エラー。スペース4つに統一する。
- `is` と `==` の混同: `is` は同一オブジェクト判定、値比較は `==`。`None` 判定だけ `is None`。
- 浅いコピー: `b = a` は同じリストを指す。独立させるなら `a.copy()` や `copy.deepcopy`。
- GIL の誤解: 通常ビルドはスレッドを増やしても CPU 計算は速くならない。CPU 並列は multiprocessing / ネイティブ / 3.13 の free-threaded（実験）へ。
- 3.13 の新機能の実験性: free-threaded と JIT は experimental。本番は通常ビルドで、互換性が固まるまで様子見が無難。
- 型ヒントは実行時に強制されない: 型を書いても実行時の値検査は別途必要（mypy 等は静的、`pydantic` 等は実行時検証）。

## 関連
- フレームワーク: [../django/](../django/)（Django）/ [../fastapi/](../fastapi/)（FastAPI）。
- 環境管理: [../環境管理.md](../環境管理.md)（venv / uv / Docker）。
- 言語概要: [../README.md](../README.md)。
