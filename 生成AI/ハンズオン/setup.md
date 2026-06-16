# setup — 環境構築（最初に1回だけ）

> ここで「Python＋各社SDK＋APIキー＋疎通確認」までやる。以降の各ハンズオンはこれが済んでいる前提で始まる。

## ゴール
`python hello.py` を実行すると Claude が1行返してくる。Gemini / OpenAI も同様に叩ける状態にする。

---

## ステップ1：Python と仮想環境
**やること**：Python 3.10+ を用意し、専用の仮想環境を作る。
**なぜ**：SDKを世界全体に入れず、このハンズオン専用に隔離するため。
```bash
python3 --version            # 3.10 以上ならOK
mkdir -p ~/genai-handson && cd ~/genai-handson
python3 -m venv .venv
source .venv/bin/activate     # Windowsは .venv\Scripts\activate
```
**成功**：プロンプト頭に `(.venv)` が付く。
**失敗の原因**：`python3` が無い → OSにPythonを入れる。`(.venv)` が出ない → `source` のパス間違い。

## ステップ2：SDKを入れる
**やること**：使う分だけインストール（Claude は必須、他は任意）。
```bash
pip install anthropic          # Claude（主軸・必須）
pip install google-genai       # Gemini（任意）
pip install openai             # ChatGPT(OpenAI)（任意）
pip install voyageai           # 埋め込み（RAG章で使う・任意）
```
**成功**：`Successfully installed ...` と出る。
**失敗の原因**：`pip` が古い → `pip install -U pip`。会社PCのプロキシ → `--proxy` 指定。

## ステップ3：APIキーを取得する
**やること**：使うサービスの管理画面でキーを発行する。
| サービス | 取得場所 | 環境変数名 |
|---|---|---|
| **Claude** | console.anthropic.com → API Keys | `ANTHROPIC_API_KEY` |
| **Gemini** | aistudio.google.com → Get API key | `GEMINI_API_KEY` |
| **ChatGPT(OpenAI)** | platform.openai.com → API keys | `OPENAI_API_KEY` |
| **Voyage**（埋め込み） | dash.voyageai.com | `VOYAGE_API_KEY` |

**なぜ**：APIは「誰が・どの請求先で」使うかをキーで識別する。課金もここに紐づく。
> どれも**支払い方法の登録（少額のクレジット購入 or カード）が必要**。無料枠がある場合もあるが、本ハンズオンは従量課金前提。

## ステップ4：キーを環境変数に入れる
**やること**：シェルにエクスポートする（コードに**書かない**）。
```bash
export ANTHROPIC_API_KEY="sk-ant-..."    # 自分のキーに置換
export GEMINI_API_KEY="..."              # 使うなら
export OPENAI_API_KEY="sk-..."           # 使うなら
export VOYAGE_API_KEY="pa-..."           # RAG章で使うなら
```
**なぜ**：キーをソースに書くと**git に乗って漏れる**（このリポジトリのセキュリティ方針）。SDKは既定でこの環境変数を読む。
**成功**：`echo $ANTHROPIC_API_KEY` で値が表示される。
**失敗の原因**：別のターミナルを開くと消える → ターミナルごとに `export` するか `~/.bashrc` に書く。`.env` 派なら `python-dotenv` を使う。

## ステップ5：疎通確認（Claude）
**やること**：`hello.py` を作って実行する。
```python
# hello.py
import anthropic

client = anthropic.Anthropic()  # ANTHROPIC_API_KEY を自動で読む

resp = client.messages.create(
    model="claude-haiku-4-5",          # 安く速い。重い用途は claude-opus-4-8
    max_tokens=100,                    # 出力上限（課金暴走防止に必須）
    messages=[{"role": "user", "content": "1行で自己紹介して"}],
)
print(resp.content[0].text)
```
```bash
python hello.py
```
**成功**：Claude の自己紹介が1行表示される。
**失敗の原因**：
- `AuthenticationError` → キーが間違い/未設定。`echo $ANTHROPIC_API_KEY` を確認。
- `NotFoundError`（モデル名）→ モデルIDのtypo。`claude-haiku-4-5` をそのまま。
- `PermissionDenied` / 残高エラー → 支払い方法/クレジット未登録。
- `content[0].text` でエラー → 応答が複数ブロックのことがある。後の章で `block.type == "text"` で絞る書き方を学ぶ。

## ステップ6（任意）：Gemini / OpenAI でも同じことを
**やること**：読み替えを確認しておく（以降は基本Claudeで進める）。
```python
# Gemini
from google import genai
g = genai.Client()  # GEMINI_API_KEY を読む
print(g.models.generate_content(model="gemini-2.5-flash",
      contents="1行で自己紹介して").text)
```
```python
# OpenAI (ChatGPT)
from openai import OpenAI
o = OpenAI()  # OPENAI_API_KEY を読む
print(o.responses.create(model="gpt-5",
      input="1行で自己紹介して").output_text)
```
> 3社で**考え方は同じ**（テキストを送ってテキストが返る）。違うのは SDK の関数名・モデルID・パラメータ名だけ。各章は Claude を主、要所で読み替えを示す。
> モデルID・料金は変動するので、実装時は各社公式で最新を確認する（Claude は `claude-api` スキル）。

## わざと壊して直す（エラー体験）
1. `export ANTHROPIC_API_KEY=""`（空にする）→ `python hello.py` → **AuthenticationError** を見る。これが「キー未設定」の見え方。正しい値に戻す。
2. `model="claude-haiku"`（不完全なID）にする → **NotFoundError**。正しい `claude-haiku-4-5` に戻す。
3. `max_tokens` を消す → SDKがエラー（必須パラメータ）。**必ず付ける**癖をここで体に入れる。

## 次へ
疎通できたら → [llm基礎.md](./llm基礎.md)
