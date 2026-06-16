# ハンズオン：LLM基礎

> 対応する理屈 → [../llm基礎.md](../llm基礎.md)。前提 → [setup.md](./setup.md) 完了。

## ゴール
最初のAPI呼び出しから、**トークン計測・温度の効き・ストリーミング・会話の継続**までを手で動かして体感する。

---

## ステップ1：1回の呼び出しを正しく書く
**やること**：応答ブロックを `type` で絞って取り出す。
```python
# step1.py
import anthropic
client = anthropic.Anthropic()

resp = client.messages.create(
    model="claude-haiku-4-5",
    max_tokens=200,
    messages=[{"role": "user", "content": "日本の首都は？1単語で。"}],
)
# content は複数ブロックの配列。text だけ取り出すのが安全
text = "".join(b.text for b in resp.content if b.type == "text")
print("回答:", text)
print("入力トークン:", resp.usage.input_tokens)
print("出力トークン:", resp.usage.output_tokens)
```
**なぜ**：`content[0].text` 直アクセスは、応答が複数ブロックだと壊れる。`usage` に**課金の実数**が乗る。
**成功**：「東京」と、入力/出力トークン数が表示される。
**失敗の原因**：`AttributeError: text` → text以外のブロックを触っている。上の絞り込みで回避。

## ステップ2：トークンを「数える」
**やること**：送る前にトークン数を見積もる（課金の前提）。
```python
# step2.py の一部
n = client.messages.count_tokens(
    model="claude-haiku-4-5",
    messages=[{"role": "user", "content": "これは日本語のトークン数を数える実験です。"*10}],
)
print("送る前の入力トークン見積り:", n.input_tokens)
```
**なぜ**：料金も上限もトークンで決まる。**文字数ではなくトークンで数える**のが鉄則（日本語は割高）。
**成功**：トークン数が表示される。`"..."*10` の倍数を変えると数が比例して増える。
**失敗の原因**：`tiktoken` 等の他社トークナイザで数える → Claudeでは数がズレる。必ず `count_tokens` を使う。

## ステップ3：温度（temperature）の効きを見る
**やること**：同じ質問を温度0と温度1で数回ずつ投げて、ブレを比べる。
```python
# step3.py
import anthropic
client = anthropic.Anthropic()

def ask(temp):
    r = client.messages.create(
        model="claude-haiku-4-5", max_tokens=60, temperature=temp,
        messages=[{"role": "user", "content": "カフェの名前を1つ提案して。名前だけ。"}],
    )
    return "".join(b.text for b in r.content if b.type == "text").strip()

print("temp=0:", [ask(0.0) for _ in range(3)])
print("temp=1:", [ask(1.0) for _ in range(3)])
```
**なぜ**：低温＝決定的（抽出/分類/JSON向き）、高温＝多様（アイデア出し向き）。**抽出タスクで高温はバグの元**だと体で分かる。
**成功**：temp=0 はほぼ同じ、temp=1 はバラける。
**失敗の原因**：`temperature` がエラー → 一部の最新モデル（Opus 4.8等）は temperature 非対応。**Haiku/Sonnetで実施**するか、温度を外して進める（読み替え：本実験はHaikuで行う）。

## ステップ4：ストリーミング（逐次表示）
**やること**：トークンを届いた順に表示してチャットの体感を作る。
```python
# step4.py
import anthropic
client = anthropic.Anthropic()

with client.messages.stream(
    model="claude-haiku-4-5", max_tokens=300,
    messages=[{"role": "user", "content": "短い俳句を1つ"}],
) as stream:
    for piece in stream.text_stream:
        print(piece, end="", flush=True)
print()
```
**なぜ**：最初のトークンまでの体感時間（TTFT）が縮む。長い出力でタイムアウトも避けられる。
**成功**：文字が**少しずつ**流れて表示される。
**失敗の原因**：一気に出る/固まる → `flush=True` 忘れ、または出力が短すぎて一瞬。`max_tokens` を増やすと流れが見える。

## ステップ5：会話を続ける（履歴は毎回全部送る）
**やること**：前のやり取りを `messages` に積んで文脈を保つ。
```python
# step5.py
import anthropic
client = anthropic.Anthropic()
msgs = [{"role": "user", "content": "私の名前はハギノです。"}]

r1 = client.messages.create(model="claude-haiku-4-5", max_tokens=80, messages=msgs)
a1 = "".join(b.text for b in r1.content if b.type == "text")
print("A1:", a1)

msgs.append({"role": "assistant", "content": a1})       # 応答を履歴に積む
msgs.append({"role": "user", "content": "私の名前は？"})  # 次の質問
r2 = client.messages.create(model="claude-haiku-4-5", max_tokens=80, messages=msgs)
print("A2:", "".join(b.text for b in r2.content if b.type == "text"))
```
**なぜ**：API は**ステートレス**。「覚えている」ように見せるには毎回全履歴を送る＝履歴が伸びるほどコスト増（→ [運用とコスト](./運用とコスト.md)）。
**成功**：A2 で「ハギノ」と答える。
**失敗の原因**：A2が名前を知らない → assistant応答を積み忘れ、または最初のuserを消している。

## わざと壊して直す
1. ステップ5で `msgs.append({"role": "assistant", ...})` を消す → A2が名前を忘れる。**履歴を送らないと記憶しない**を体感。戻す。
2. `max_tokens=5` にして長文を頼む → 途中で切れる（`stop_reason == "max_tokens"`）。**上限不足の見え方**。増やす。

## 次へ
→ [プロンプト設計.md](./プロンプト設計.md)
