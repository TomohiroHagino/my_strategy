# ハンズオン：RAG（検索拡張生成）

> 対応する理屈 → [../rag.md](../rag.md)。前提 → [setup.md](./setup.md)。

## ゴール
**自前の文書を埋め込み→検索→プロンプトに注入**して、「Claudeが知らない自社情報」に根拠つきで答えさせる最小RAGを作る。ベクトルDBは使わず**メモリ上のリスト＋コサイン類似度**で本質だけ体験する。

> 埋め込み（embedding）は Anthropic に専用APIが無いため、**Anthropic推奨の Voyage AI** を使う（`pip install voyageai`、`VOYAGE_API_KEY`）。OpenAI(`text-embedding-3-small`) / Gemini でも考え方は同じ。生成は Claude。

---

## ステップ1：知識ベースを用意（チャンク）
**やること**：Claudeが知らない架空の社内ルールを数件、短いチャンクで持つ。
```python
# rag.py（以降ここに追記していく）
DOCS = [
    "有給は最大20日まで翌年度に繰り越せる。21日目以降は消滅する。",
    "リモート勤務は週3日まで。上長の事前申請が必要。",
    "経費精算は毎月25日締め、翌月10日払い。領収書は電子提出可。",
    "社用PCの私的利用は禁止。違反は就業規則第12条に基づき処分対象。",
]
```
**なぜ**：RAGは「検索単位＝チャンク」の質が命。ここでは1ルール=1チャンクと意味の境界で切っている。
**成功**：リストが定義できればOK。

## ステップ2：埋め込みでベクトル化
**やること**：各チャンクをベクトルに変換する。
```python
import voyageai
vo = voyageai.Client()  # VOYAGE_API_KEY を読む

def embed(texts, input_type):
    # input_type は "document"（文書側）/ "query"（質問側）を分けるのが定石
    return vo.embed(texts, model="voyage-3", input_type=input_type).embeddings

DOC_VECS = embed(DOCS, "document")
print("ベクトル次元:", len(DOC_VECS[0]), " 文書数:", len(DOC_VECS))
```
**なぜ**：意味が近い文はベクトルが近い＝キーワード一致でなく**意味で検索**できる。文書側と質問側で同じモデルを使う。
**成功**：次元数（数百〜千）と文書数が表示される。
**失敗の原因**：`VOYAGE_API_KEY` 未設定 → setupで取得。別の埋め込みモデルに変えたら**全文書を作り直す**（混ぜると検索が壊れる）。

## ステップ3：質問に近いチャンクを検索（top-k）
**やること**：コサイン類似度で上位kを取る。
```python
import numpy as np

def cosine(a, b):
    a, b = np.array(a), np.array(b)
    return float(a @ b / (np.linalg.norm(a) * np.linalg.norm(b)))

def search(question, k=2):
    qv = embed([question], "query")[0]
    scored = sorted(((cosine(qv, dv), doc) for dv, doc in zip(DOC_VECS, DOCS)),
                    reverse=True)
    return scored[:k]

for s, d in search("有給って繰り越せる？何日まで？"):
    print(round(s, 3), d)
```
**なぜ**：質問もベクトル化し、近い文書を引く。これがRAGの「検索」。
**成功**：有給のチャンクが最上位に来る（言い換え「繰り越せる？」でもヒット）。
**失敗の原因**：無関係なチャンクが上位 → チャンクが雑/質問が曖昧。固有名詞が要るならハイブリッド検索（キーワード併用）を足す（理屈は[../rag.md](../rag.md)）。

## ステップ4：取れた文書を文脈に注入してClaudeに答えさせる
**やること**：検索結果を `--- 文脈 ---` で囲んで渡し、出典つきで答えさせる。
```python
import anthropic
client = anthropic.Anthropic()

def answer(question):
    hits = search(question, k=2)
    context = "\n".join(f"[{i+1}] {doc}" for i, (_, doc) in enumerate(hits))
    prompt = f"""あなたは社内アシスタント。次の文脈情報"だけ"に基づいて答えること。
文脈に答えが無ければ「分かりません」と言う（推測で補わない）。根拠にした番号[n]を必ず示す。
--- 文脈ここから ---
{context}
--- 文脈ここまで ---
質問: {question}"""
    r = client.messages.create(model="claude-haiku-4-5", max_tokens=300, temperature=0,
        messages=[{"role": "user", "content": prompt}])
    return "".join(b.text for b in r.content if b.type == "text")

print(answer("有給は何日繰り越せる？"))
```
**なぜ**：「文脈だけに基づく」「無ければ分からないと言う」「出典[n]」が**ハルシネーション抑制の核**。
**成功**：「最大20日まで繰り越せます[1]」のように**根拠番号つき**で答える。
**失敗の原因**：作話する → 「文脈に無ければ分からないと言え」を必ず入れる。出典が出ない → プロンプトで明示。

## ステップ5：RAGの限界を見る（知らないことを聞く）
**やること**：知識ベースに無い質問をして挙動を見る。
```python
print(answer("出張の日当はいくら？"))   # DOCSに無い
```
**なぜ**：**検索が外す＝LLMも答えられない**。RAGの失敗の多くは生成でなく検索側。
**成功**：「分かりません」と正直に言う（作話しない）。
**失敗の原因**：それっぽい嘘を作る → ガード文を強める／文書を足す。

## わざと壊して直す
1. ステップ2の文書側 `input_type` を `"query"` に変えてしまう → 検索精度が落ちる。**文書側は"document"**に戻す。
2. ステップ4のプロンプトから「文脈だけ／無ければ分からない」を消す → Claudeが一般知識で答え始める（社内ルールと食い違う）。戻す。

## 片付け / 発展
- 今回はメモリ上のリスト。実務は**ベクトルDB**に保存（PostgreSQL拡張 pgvector が手軽：[../../データベース/postgresql.md](../../データベース/postgresql.md)）。
- 精度を上げる次の一手：**ハイブリッド検索・再ランク・メタデータフィルタ**（→ [../rag.md](../rag.md)）。

## 次へ
→ [エージェント.md](./エージェント.md)
