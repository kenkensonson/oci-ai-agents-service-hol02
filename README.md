# 本ハンズオンの概要
本ハンズオンは株価情報を取得するAIエージェントをADKで構築します。株価情報を取得する処理にはMCP(Model Context Protocol)を利用し、ADKとMCPのシンプルな連携処理を学習します。また、MCPで株価情報を取得する際は[Stooq](https://stooq.com/)のサービスを利用します。

# 構成と手順
本ハンズオンは[こちらのハンズオン](https://github.com/kenkensonson/oci-ai-agents-service-hol01)の実施が既に完了している前提となります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/109260/8c27a1df-b374-4fa9-8e6d-dd2b323ef49b.png)


手順の概要は以下の通りです。

1. OCI AI Agent Serviceのコンソールからエージェントのエンドポイントを作成([こちらのハンズオン](https://github.com/kenkensonson/oci-ai-agents-service-hol01)で実施済)
2. Python環境の構築およびAgent Development Kitの環境を構築([こちらのハンズオン](https://github.com/kenkensonson/oci-ai-agents-service-hol01)で実施済)
3. Stooqから株価情報を取得するMCP Server(mpc_server.py)を作成します。
4. OCI AI Agents ServiceのエージェントとMCPを連動させるため、MCP Client(mcp_client.py)を作成します。
5. 処理の実行

上記手順1と2については実施済みの前提として、以降は3と4の手順となります。

# MCP Server(mcp_server.py)の実装

まず必要なライブラリをインストールします。

```python
pip install fastmcp pandas
```

次に、Stooqから株価情報を取得するMCP Serverを作成します。下記コードをmcp_server.pyとして保存します。

```python
# 以下のコードをmcp_server.pyとして保存

from mcp.server.fastmcp import FastMCP
from typing import Annotated
import yfinance as yf  # 不要なら削除してもOK
import pandas as pd

# MCPサーバーインスタンスの作成
mcp = FastMCP("StockData")

# MCPツールとして Stooq を使った株価取得関数を登録
@mcp.tool()
def get_stock_stooq(
    ticker: Annotated[str, "Stooq形式のティッカー（例: 'aapl.us', '7203.jp'）"],
    start_date: Annotated[str, "開始日 (例: '2024-01-01')"],
    end_date: Annotated[str, "終了日 (例: '2024-12-31')"]
) -> str:
    """
    Stooqから株価データを取得し、指定された日付範囲で返す。
    """
    url = f"https://stooq.com/q/d/l/?s={ticker.lower()}&i=d"
    df = pd.read_csv(url)

    if df.empty or "Date" not in df.columns:
        return f"{ticker} のデータが取得できませんでした。"

    df["Date"] = pd.to_datetime(df["Date"])
    filtered_df = df[(df["Date"] >= start_date) & (df["Date"] <= end_date)]

    if filtered_df.empty:
        return f"{ticker} の {start_date} 〜 {end_date} のデータは見つかりませんでした。"

    return filtered_df.to_string(index=False)

# サーバー起動（HTTP ストリームモード）
if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```

# MCP Client(mcp_client.py)の実装
次に、MCP Clientを作成します。下記のコードをmcp_client.pyという名前で保存します。

- <エージェントのエンドポイントのOCID>についてはご自身のOCIDに変更してください。
- input_messageが入力プロンプトになりますので、任意の入力を定義してください。

```python
import asyncio

from mcp.client.session_group import StreamableHttpParameters
from oci.addons.adk import Agent, AgentClient
from oci.addons.adk.mcp import MCPClientStreamableHttp

async def main():

    params = StreamableHttpParameters(
        url="http://localhost:8000/mcp",
    )

    async with MCPClientStreamableHttp(
        params=params,
        name="Streamable MCP Server",
    ) as mcp_client:

        client = AgentClient(
            auth_type="api_key",
            profile="DEFAULT",
            region="us-chicago-1"
        )

        agent = Agent(
            client=client,
            agent_endpoint_id="<エージェントのエンドポイントのOCID>",
            instructions="""
            あなたは株価情報を取得するエージェントです。下記の手順で株価を取得してユーザーに応答してください。

            1. 入力プロンプトの中の企業名をStooqのティッカーシンボルに変換します。(例：apple -> aapl.us)
            2. 変換したティッカーシンボルを株価取得関数の引数に入力します。
            """,
            tools=[await mcp_client.as_toolkit()],
        )

        agent.setup()

        # Should trigger the `add` tool
        input_message = "2025年7月25日のoracleの株価の終値は？"
        print(f"Running: {input_message}")
        response = await agent.run_async(input_message)
        response.pretty_print()

if __name__ == "__main__":
    asyncio.run(main())
```

# 処理の実行

ターミナルから下記コマンドを実行し、まずはMCP serverを起動します。

```python
$ python mcp_server.py 
```

エラーが表示されず以下のような出力になればMCPサーバーが正常動作しています。

```python
$ python mcp_server.py 
INFO:     Started server process [11819]
INFO:     Waiting for application startup.
[07/29/25 07:47:36] INFO     StreamableHTTP session manager started                                                                                         streamable_http_manager.py:109
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
```

次に別のターミナルから下記コマンドを実行しエージェントを起動します。これにより、入力プロンプトからエージェントが株価情報を取得し、最終的な応答テキストをクライアントに返す処理が実行されます。

```python
$ python mcp_client.py 
```

エラーが表示されず下記のような出力になれば処理が正常に完了しています。

```python
$ python mcp_agent_stooq.py 
[07/29/25 07:43:06]  INFO     Checking integrity of agent details...                                                                                                                      
                     INFO     Checking synchronization of local and remote agent settings...                                                                                              
                     INFO     Agent settings are not synchronized. Updating remote agent settings                                                                                         
[07/29/25 07:43:11]  INFO     Waiting for agent ocid1.genaiagent.oc1.us-chicago-1.amaaaaaal7l2mtaadca2hwhfoyxyxibd4cpteibxwljwhtlgt7zqwtumujnq to be active...                            
[07/29/25 07:43:16]  INFO     Waiting for agent ocid1.genaiagent.oc1.us-chicago-1.amaaaaaal7l2mtaadca2hwhfoyxyxibd4cpteibxwljwhtlgt7zqwtumujnq to be active...                            
[07/29/25 07:43:22]  INFO     Waiting for agent ocid1.genaiagent.oc1.us-chicago-1.amaaaaaal7l2mtaadca2hwhfoyxyxibd4cpteibxwljwhtlgt7zqwtumujnq to be active...                            
[07/29/25 07:43:27]  INFO     Waiting for agent ocid1.genaiagent.oc1.us-chicago-1.amaaaaaal7l2mtaadca2hwhfoyxyxibd4cpteibxwljwhtlgt7zqwtumujnq to be active...                            
[07/29/25 07:43:32]  INFO     Waiting for agent ocid1.genaiagent.oc1.us-chicago-1.amaaaaaal7l2mtaadca2hwhfoyxyxibd4cpteibxwljwhtlgt7zqwtumujnq to be active...                            
[07/29/25 07:43:37]  INFO     Waiting for agent ocid1.genaiagent.oc1.us-chicago-1.amaaaaaal7l2mtaadca2hwhfoyxyxibd4cpteibxwljwhtlgt7zqwtumujnq to be active...                            
                     INFO     Checking synchronization of local and remote function tools...                                                                                              
╭─ Local and remote function tools found ─╮
│ Local function tools (1):               │
│ ['get_stock_stooq']                     │
│                                         │
│ Remote function tools (1):              │
│ ['get_stock_stooq']                     │
╰─────────────────────────────────────────╯
                     INFO     Checking synchronization of local and remote RAG tools...                                                                                                   
Running: 2025年7月25日のoracleの株価の終値は？
╭───────────────────────────────── Chat request to remote agent: None ──────────────────────────────────╮
│ (Local --> Remote)                                                                                    │
│                                                                                                       │
│ user message:                                                                                         │
│ 2025年7月25日のoracleの株価の終値は？                                                                 │
│                                                                                                       │
│ performed actions by client:                                                                          │
│ []                                                                                                    │
│                                                                                                       │
│ session id:                                                                                           │
│ ocid1.genaiagentsession.oc1.us-chicago-1.amaaaaaa7mjirbaayiuunnl6647f6srjcdzsymoghb3t4pn2u6ijf4q6vfkq │
╰───────────────────────────────────────────────────────────────────────────────────────────────────────╯
╭───────────────────────────────────────── Chat response from remote agent ──────────────────────────────────────────╮
│ (Local <-- Remote)                                                                                                 │
│                                                                                                                    │
│ agent message:                                                                                                     │
│ null                                                                                                               │
│                                                                                                                    │
│ required actions for client to take:                                                                               │
│ [                                                                                                                  │
│     {                                                                                                              │
│         "action_id": "807a71ae-60e5-4903-9258-9ba6c62bea26",                                                       │
│         "required_action_type": "FUNCTION_CALLING_REQUIRED_ACTION",                                                │
│         "function_call": {                                                                                         │
│             "name": "get_stock_stooq",                                                                             │
│             "arguments": "{\"ticker\": \"orcl.us\", \"start_date\": \"2025-07-25\", \"end_date\": \"2025-07-25\"}" │
│         }                                                                                                          │
│     }                                                                                                              │
│ ]                                                                                                                  │
│                                                                                                                    │
│ guardrail result:                                                                                                  │
│ None                                                                                                               │
│                                                                                                                    │
│                                                                                                                    │
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
╭──── Function call requested by agent and mapped local handler function ─────╮
│ Agent function tool name:                                                   │
│ get_stock_stooq                                                             │
│                                                                             │
│ Agent function tool call arguments:                                         │
│ {'ticker': 'orcl.us', 'start_date': '2025-07-25', 'end_date': '2025-07-25'} │
│                                                                             │
│ Mapped local handler function name:                                         │
│ invoke_mcp_tool_proxy                                                       │
╰─────────────────────────────────────────────────────────────────────────────╯
╭───────────────────────────────────────────────────────────────────────────────── MCP tool call result ─────────────────────────────────────────────────────────────────────────────────╮
│ MCP tool name:                                                                                                                                                                         │
│ get_stock_stooq                                                                                                                                                                        │
│                                                                                                                                                                                        │
│ MCP tool args:                                                                                                                                                                         │
│ {'ticker': 'orcl.us', 'start_date': '2025-07-25', 'end_date': '2025-07-25'}                                                                                                            │
│                                                                                                                                                                                        │
│ MCP tool call result:                                                                                                                                                                  │
│ {"meta":null,"content":[{"type":"text","text":"      Date   Open   High    Low  Close    Volume\n2025-07-25 242.34 245.47 241.43 245.12                                                │
│ 7149571.0","annotations":null}],"isError":false}                                                                                                                                       │
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
╭─────────────────────────────────────────────────── Obtained local function execution result ───────────────────────────────────────────────────╮
│ {"type":"text","text":"      Date   Open   High    Low  Close    Volume\n2025-07-25 242.34 245.47 241.43 245.12 7149571.0","annotations":null} │
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
╭────────────────────────────────────────────────────────────────────────── Chat request to remote agent: None ──────────────────────────────────────────────────────────────────────────╮
│ (Local --> Remote)                                                                                                                                                                     │
│                                                                                                                                                                                        │
│ user message:                                                                                                                                                                          │
│ null                                                                                                                                                                                   │
│                                                                                                                                                                                        │
│ performed actions by client:                                                                                                                                                           │
│ [{'action_id': '807a71ae-60e5-4903-9258-9ba6c62bea26', 'performed_action_type': 'FUNCTION_CALLING_PERFORMED_ACTION', 'function_call_output': '{"type":"text","text":"      Date   Open │
│ High    Low  Close    Volume\\n2025-07-25 242.34 245.47 241.43 245.12 7149571.0","annotations":null}'}]                                                                                │
│                                                                                                                                                                                        │
│ session id:                                                                                                                                                                            │
│ ocid1.genaiagentsession.oc1.us-chicago-1.amaaaaaa7mjirbaayiuunnl6647f6srjcdzsymoghb3t4pn2u6ijf4q6vfkq                                                                                  │
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
╭───────────────────────────────────────────── Chat response from remote agent ──────────────────────────────────────────────╮
│ (Local <-- Remote)                                                                                                         │
│                                                                                                                            │
│ agent message:                                                                                                             │
│ {                                                                                                                          │
│     "role": "AGENT",                                                                                                       │
│     "content": {                                                                                                           │
│         "text": "2025\u5e747\u670825\u65e5\u306eOracle\u306e\u682a\u4fa1\u306e\u7d42\u5024\u306f245.12\u3067\u3059\u3002", │
│         "citations": null,                                                                                                 │
│         "paragraph_citations": null                                                                                        │
│     },                                                                                                                     │
│     "time_created": "2025-07-29T07:43:44.469000+00:00"                                                                     │
│ }                                                                                                                          │
│                                                                                                                            │
│ required actions for client to take:                                                                                       │
│ null                                                                                                                       │
│                                                                                                                            │
│ guardrail result:                                                                                                          │
│ None                                                                                                                       │
│                                                                                                                            │
│                                                                                                                            │
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
╭────────────── Agent run response ───────────────╮
│ agent text message:                             │
│ 2025年7月25日のOracleの株価の終値は245.12です。    │
╰─────────────────────────────────────────────────╯
```

最終的に株価情報が取得されていることがわかります。

以上、ADKで構築したエージェントシステムからMCPを利用するハンズオンでした。
