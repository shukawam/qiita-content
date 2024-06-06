---
title: OCI Generative AI Service(Cohere Command R)を用いたTool Use(Function Calling)
tags:
  - "oci"
  - "Cohere"
private: false
updated_at: ""
id: null
organization_url_name: oracle
slide: false
ignorePublish: false
---

# はじめに

6/4 に OCI Generative AI Service は Cohere 社の [Command R](https://docs.cohere.com/docs/command-r) が使えるようになりました。

https://docs.oracle.com/en-us/iaas/releasenotes/changes/fc743a75-4cbb-4ffe-b9a1-d5c4af3dc4b4/

Command R は日本語を含む多言語対応のモデルであり、RAG[^1]と Tool Use(a.k.a Function Calling)にて高い精度とパフォーマンス（低レイテンシー、高スループット）を実現しているモデルのようです。
Cohere 社のドキュメントでは、最大コンテキスト長が 128k で最大の出力トークンが 4k と記載がありますが、OCI のサービスとして使う場合は最大コンテキスト長が 16k で最大の出力トークンが 4k となっていますので注意が必要です。とはいえ、待望の日本語生成対応のモデルが OCI でも使えるようになったので、この記事では OCI Generative AI Service で Command R の Tool Use を試してみようと思います。

[^1]: RAG: Retrieval Augumented Generation(検索拡張生成)

# Tool Use とは？

Tool Use(Function Calling とも呼ばれる)は、LLM の機能を補強するために外部のツールと接続することを言います。Cohere 社のドキュメントでは、Single-Step Tool Use(Function Calling)と Multi-Step Tool Use(Agents)と分類されており、それぞれ以下のように振る舞います。

- Single-Step Tool Use(Function Calling): 提供されたツールセットからどのツールをどのようなパラメータを渡して実行すれば良いかを LLM に選択してもらう
- Multi-Step Tool Use(Agents): 提供されたツールセットからどのツールをどの順番でどのようなパラメータを渡して実行すれば良いかを LLM に決定してもらう

# 実際に試してみる

## 前提

本記事で試す環境は、以下の前提で動作確認をしています。

- Python: 3.11.7
- OCI SDK for Python 2.128.0

共通的に必要になるモジュールのインポートやクライアントの初期化処理は以下のように実施しています。

```py
import os
from dotenv import load_dotenv

from oci.auth.signers import InstancePrincipalsSecurityTokenSigner
from oci.generative_ai_inference.generative_ai_inference_client import GenerativeAiInferenceClient
from oci.generative_ai_inference.models import (
    ChatDetails,
    OnDemandServingMode,
    CohereChatRequest,
    CohereTool,
    CohereParameterDefinition,
    CohereToolCall,
    CohereToolResult
)

load_dotenv()
compartment_id = os.getenv("COMPARTMENT_ID")
endpoint = os.getenv("INFERENCE_ENDPOINT")
signer = InstancePrincipalsSecurityTokenSigner()
generative_ai_inference_client = GenerativeAiInferenceClient(config = {}, signer = signer, service_endpoint = endpoint)
```

Notebook 全体は以下で公開しています。

https://github.com/shukawam/notebooks/blob/main/oci/postal-code.ipynb

## Single-Step Tool Use

Single-Step Tool Use を試してみます。外部ツールとしては、無料で公開されている[郵便番号 API](https://github.com/ttskch/jp-postal-code-api) を扱います。まずは、外部ツールなしでモデルの出力を確認してみます。

```py
response = generative_ai_inference_client.chat(
    chat_details=ChatDetails(
        compartment_id=compartment_id,
        serving_mode=OnDemandServingMode(
            model_id="cohere.command-r-16k"
        ),
        chat_request=CohereChatRequest(
            message="107-0061の郵便番号の住所を教えてください",
            max_tokens=200,
        )
    )
)

print(f"response text: {response.data.chat_response.text}")
```

実行結果

```py
response text: 〒107-0061 東京都港区六本木7丁目11番 周辺です。
```

東京都港区六本木 7 丁目 11 番周辺の郵便番号は、調べてみると 106-0032 であることが確認できるので、LLM がハルシネーションを起こしていることがわかります。
これを改善するために、郵便番号を検索する API を使うツールを実装してみましょう。

```py
import requests
import re

def query_address_from_postal_code(postal_code: str) -> dict:
    if re.match("^[0-9]{3}-[0-9]{4}$", postal_code):
        postal_code = postal_code.replace("-", "")
    headers = {"content-type": "application/json"}
    response = requests.get(f"https://jp-postal-code-api.ttskch.com/api/v1/{postal_code}.json", headers=headers)
    data_ja = response.json()["addresses"][0]["ja"]
    if response.status_code == 200:
        return {
            "address": f"{data_ja['prefecture']}{data_ja['address1']}{data_ja['address2']}"
        }
    else:
        return {
            "address": "与えられた郵便番号に合致する住所は見つかりませんでした"
        }
```

上記のように与えられた郵便番号に対して、日本語で市町村レベルまで返却するように関数を実装してみました。実行すると、以下のような辞書型が返却されます。

```py
{'address': '東京都港区北青山'}
```

この関数を OCI Generative AI Service(Cohere Command R) で使えるようにツールとして登録します。配列で定義してある通り、複数のツールを登録することができますが、ここでは簡単のために住所検索 API を使う関数を 1 つ登録します。

```py
tools = [
    CohereTool(
        name = "query_address_from_postal_code",
        description = "郵便番号を元に住所を検索します",
        parameter_definitions = {
            "postal_code": CohereParameterDefinition(
                description = "この郵便番号を元に住所を検索します。",
                type = "str",
                is_required = True
            )
        }
    )
]
```

どのツールをどのようなパラメータで使えば良いかを LLM に判断させるために、チャットのリクエストに `tools` を含めます。また、LLM が与えられたツールを活用し、回答を作成するために preamble[^2] を上書きします。

[^2]: モデルの全体的な動作と会話スタイルを調整するために使用されるプロンプトの一部

```py
preamble_override = """
## タスクとコンテキスト
人々の質問やその他のリクエストにインタラクティブに答える手助けをする。あらゆる種類のトピックについて、非常に幅広い質問を受けます。幅広い検索エンジンや類似のツールが用意されており、それらを使って答えを調べます。あなたは、ユーザーのニーズにできる限り応えることに集中すべきです。

## スタイルガイド
ユーザーから別の回答スタイルを要求されない限り、適切な文法とスペルを使い、完全な文章で回答してください。
"""

response = generative_ai_inference_client.chat(
    chat_details=ChatDetails(
        compartment_id=compartment_id,
        serving_mode=OnDemandServingMode(
            model_id="cohere.command-r-16k"
        ),
        chat_request=CohereChatRequest(
            message="107-0061の郵便番号の住所を教えてください",
            max_tokens=200,
            tools=tools,
            preamble_override=preamble_override
        )
    )
)

print(response.data)
```

実行結果

```json
{
  "chat_response": {
    "api_format": "COHERE",
    "chat_history": [
      {
        "message": "107-0061\u306e\u90f5\u4fbf\u756a\u53f7\u306e\u4f4f\u6240\u3092\u6559\u3048\u3066\u304f\u3060\u3055\u3044",
        "role": "USER"
      },
      {
        "message": "I will search for the address using the postal code 107-0061.",
        "role": "CHATBOT",
        "tool_calls": [
          {
            "name": "query_address_from_postal_code",
            "parameters": {
              "postal_code": "107-0061"
            }
          }
        ]
      }
    ],
    "citations": null,
    "documents": null,
    "error_message": null,
    "finish_reason": "COMPLETE",
    "is_search_required": null,
    "prompt": null,
    "search_queries": null,
    "text": "I will search for the address using the postal code 107-0061.",
    "tool_calls": [
      {
        "name": "query_address_from_postal_code",
        "parameters": {
          "postal_code": "107-0061"
        }
      }
    ]
  },
  "model_id": "cohere.command-r-16k",
  "model_version": "1.2"
}
```

`$.chat_response.tool_calls` をみるとわかりますが、LLM によって使用するツールとそれを実行するためのパラメータが返却されるので、指示に従ってツールを実行します。

```py
# Map としてツールを保持しておけば複数ツールが存在する場合でも汎用的に使用可能
functions_map = {
    "query_address_from_postal_code": query_address_from_postal_code
}

tool_results = []
# $.chat_response.tool_calls に含まれるツール名とパラメータを使って、ツールを実行
for tool_call in response.data.chat_response.tool_calls:
    output = functions_map[tool_call.name](**tool_call.parameters)
    outputs = [output]
    tool_results.append(
        CohereToolResult(
            call=CohereToolCall(
                name=tool_call.name,
                parameters=tool_call.parameters
            ),
            outputs=outputs
        )
    )

print(tool_results)
```

実行結果

```py
[{
  "call": {
    "name": "query_address_from_postal_code",
    "parameters": {
      "postal_code": "107-0061"
    }
  },
  "outputs": [
    {
      "address": "\u6771\u4eac\u90fd\u6e2f\u533a\u5317\u9752\u5c71"
    }
  ]
}]
```

最後に、ツールの実行結果を LLM のテキスト生成に使うために `tool_results` を設定し、Single-Step Tool Use を強制するために `is_force_single_step` を有効化します。

```py
response = generative_ai_inference_client.chat(
    chat_details=ChatDetails(
        compartment_id=compartment_id,
        serving_mode=OnDemandServingMode(
            model_id="cohere.command-r-16k"
        ),
        chat_request=CohereChatRequest(
            message="107-0061の住所を教えてください。",
            chat_history=response.data.chat_response.chat_history,
            max_tokens=200,
            is_force_single_step=True,
            tools=tools,
            tool_results=tool_results,
            preamble_override=preamble_override
        )
    )
)

print(response.data.chat_response.text)
```

実行結果

```py
107-0061の住所は、東京都港区北青山です。
```

外部ツールを活用し、回答が作成されていることが確認できました。

# 終わりに

PaaS で提供されるような Agent サービスが対応していないデータソース、API 呼び出しの対応や外部ツール呼び出しのログ、トレースなどを取得したい場合は重宝する機能かと思います。また、Multi-Step Tool Use は今回試せていないため、別の機会で実施し記事にしてみたいと思います。

# 参考

https://docs.cohere.com/docs/tool-usehttps://docs.cohere.com/docs/tool-use

https://github.com/shukawam/notebooks/blob/main/oci/postal-code.ipynb
