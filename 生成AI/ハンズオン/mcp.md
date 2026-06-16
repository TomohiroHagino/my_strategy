# ハンズオン：MCP（Model Context Protocol）

> 対応する理屈 → [../mcp.md](../mcp.md)。前提 → [setup.md](./setup.md)＋[エージェント.md](./エージェント.md)（tool use を理解済み）。

## ゴール
**MCPサーバを1個自作**し、Claude にその MCP ツールを使わせる。前章の「ツールを毎回手で書く」を、MCPで**標準化された接続**に置き換える体験をする。

> MCP は「ツールをどう公開・接続するか」の標準。前章の関数を **MCPサーバ**として切り出すと、対応クライアント（IDE・自作エージェント等）から共通の作法で使い回せる。

---

## ステップ1：MCPのPython SDKを入れる
**やること**：公式SDKを入れる。
```bash
pip install "mcp[cli]"      # MCP の Python SDK
```
**成功**：インストール完了。
**失敗の原因**：Python 3.10未満 → 上げる。

## ステップ2：MCPサーバを作る（ツールを公開）
**やること**：天気ツールを1つ持つMCPサーバを書く。
```python
# weather_server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("weather")           # サーバ名

@mcp.tool()
def get_weather(city: str) -> str:
    """指定した都市の天気を返す。天気を聞かれたとき使う。"""
    fake = {"Tokyo": "晴れ 23℃", "Osaka": "くもり 21℃"}
    return fake.get(city, f"{city} のデータは無い")   # 本来はAPIを叩く

if __name__ == "__main__":
    mcp.run(transport="stdio")     # ローカルは標準入出力でやり取り
```
**なぜ**：`@mcp.tool()` を付けた**普通の関数を公開するだけ**。説明文（docstring）が tool use 同様に呼び出し精度を決める。
**成功**：構文エラーなく保存できる（単体ではまだ動かさない）。
**失敗の原因**：docstringを書かない → クライアントが用途を判断しにくい。必ず書く。

## ステップ3：クライアントからサーバを起動して一覧を見る
**やること**：サーバを子プロセスとして起動し、ツール一覧を取得する。
```python
# client_list.py
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
    params = StdioServerParameters(command="python", args=["weather_server.py"])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()                 # 能力交渉
            tools = await session.list_tools()
            for t in tools.tools:
                print("ツール:", t.name, "—", t.description)

asyncio.run(main())
```
**なぜ**：MCPは **Host(クライアント) ↔ Server** をJSON-RPCで繋ぐ。ホストが起動・一覧取得・呼び出しを握る（権限管理をホストに集約できる）。
**成功**：`ツール: get_weather — ...` が表示される。
**失敗の原因**：パスエラー → `weather_server.py` と同じディレクトリで実行。固まる → サーバ側の例外。サーバを単体起動して構文を確認。

## ステップ4：MCPツールを Claude に使わせる
**やること**：MCPの一覧をClaudeのtool定義に変換し、前章のループで呼ぶ。
```python
# client_claude.py
import asyncio, anthropic
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

client = anthropic.Anthropic()

async def main():
    params = StdioServerParameters(command="python", args=["weather_server.py"])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            mtools = (await session.list_tools()).tools
            # MCPのツール定義 → Claudeのtool定義へ変換
            tools = [{"name": t.name, "description": t.description,
                      "input_schema": t.inputSchema} for t in mtools]

            msgs = [{"role": "user", "content": "東京の天気は？"}]
            for _ in range(5):
                r = client.messages.create(model="claude-haiku-4-5", max_tokens=400,
                    tools=tools, messages=msgs)
                if r.stop_reason != "tool_use":
                    print("回答:", "".join(b.text for b in r.content if b.type == "text")); break
                msgs.append({"role": "assistant", "content": r.content})
                results = []
                for b in r.content:
                    if b.type == "tool_use":
                        out = await session.call_tool(b.name, b.input)   # ★MCP経由で実行
                        text = out.content[0].text if out.content else ""
                        results.append({"type": "tool_result", "tool_use_id": b.id, "content": text})
                msgs.append({"role": "user", "content": results})

asyncio.run(main())
```
**なぜ**：前章は実行を自前の関数でやったが、ここでは **`session.call_tool` 経由でMCPサーバが実行**する。ツールの実体をサーバ側に追い出せる＝**再利用・差し替え**ができる。
**成功**：`回答: 東京は晴れ 23℃ です` のように、MCPサーバの結果を踏まえて答える。
**失敗の原因**：`inputSchema` のキー違い → SDKのフィールド名はそのまま使う。`tool_use_id` 不一致 → 前章と同じく1対1で返す。

## ステップ5：差し替えの旨味を体感
**やること**：`get_weather` の中身を別実装に変える（例：実APIを叩く/別都市を足す）だけで、クライアント側は**無変更**で動くことを確認。
**なぜ**：MCPの価値は**接続の標準化＝中身を差し替えてもクライアントは触らない**こと。1度サーバ化すれば複数ホストから使い回せる。
**成功**：サーバだけ直してクライアント再実行 → 新挙動になる。

## わざと壊して直す
1. ステップ4で **素性不明のMCPサーバを無検証で繋ぐ**想定を考える → MCPサーバは任意コード実行・データ送出の経路になりうる。**信頼できるサーバだけ繋ぐ／書き込み系は承認**（[../mcp.md](../mcp.md) のセキュリティ節）。
2. サーバの docstring を空にする → Claudeがツールを呼ばない/誤用。説明文を戻す。

## 片付け / 注意
- MCPサーバに渡す認証情報は環境変数/シークレットで。ログに残さない。
- リモート公開する場合は stdio でなく **Streamable HTTP**（[../mcp.md](../mcp.md)）。

## 次へ
→ [評価と監視.md](./評価と監視.md)
