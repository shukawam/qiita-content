---
title: Generative AI AgentsをSDKから操作する
tags:
  - "llm"
  - "oci"
  - "oracle"
private: false
updated_at: ""
id: null
organization_url_name: oracle
slide: false
ignorePublish: false
---

# はじめに

9/25 に OCI Generative AI Agents(以下、OCI Agents) が一般利用可能となりました。

https://docs.oracle.com/en-us/iaas/releasenotes/generative-ai-agents/genai-agents-GA-release.htm

Beta 版では、ナレッジベースとして OCI Search with OpenSearch のみが利用可能であったり、チャットや検索結果を保持しておくためのメモリ機構として OCI Cache が必要でしたが、GA 版となり Oracle Database 23ai, Object Storage がナレッジベースとして新たにサポートされたり、OCI Cache を構築する必要がなくなりました。また、Object Storage 限定となりますが PDF や Word といったドキュメントデータを取り込むためのデータパイプラインが整備されたり、ハイブリッド検索[^1]がサポートされたりと機能強化されて登場しました。本記事では、実際に OCI Agents を使ってアプリケーションを開発するために必要となる SDK の使い方を解説します。

[^1]: ここでは、キーワードによる検索と意味に基づく類似検索を組み合わせた検索手法を指す

## OCI Generative AI Agents は何を楽にしてくれるのか？

LLM を活用したアプリケーションを実装する際は、LLM の学習データに含まれていないようなコンテキストを LLM の出力に含めるためにファインチューニングや RAG(Retrieval-Augmented Generation)などに代表されるプロンプトエンジニアリングを用いることがあります。特に RAG の構成を作る際には、検索コンポーネントとの手続きやその検索結果をプロンプトに拡張するための手続き、前の文脈を踏まえた上で出力をさせたり、複数ユーザーの入力が混在しないようにセッション管理を行う必要があります。この手の LLM アプリケーションの開発を楽にするための抽象化を提供するフレームワーク、ライブラリ、ツールとしては、LangChain, LlamaIndex, Dify などが有名かと思います。OCI Agents は OCI 環境に特化した LLM アプリケーションの開発を容易に行うためのツール（OCI サービス）とお考えください。

## OCI SDK から OCI Generative AI Agents を操作する

ここでは、Object Storage に PDF や Word 等のドキュメントが事前に格納されており、ナレッジベースとして適切に設定されていることを前提とします。これらの手順については、以下の記事をご参照ください。

https://qiita.com/msasakaw/items/219c58ecdf98b4bfb743

今回、ドキュメントとしては以下のテキストデータを使用しています。

```txt
スパイスカレーのレシピです。

まずは、以下の材料を用意します。なお、材料は2人分です。

- 鶏モモ肉皮なし: 200g
- たまねぎ1/2: 110g
- にんにく: 4g（チューブ8㎝）
- しょうが: 4g（チューブ8㎝）
- 塩: 2g
- オリーブ油: 8ｇ
- トマト缶1/3: 60g
- カレー粉: 5g
- ガラムマサラ: 1g
- ケチャップ: 18g
- ウスターソース: 18g

作り方は以下の通りです。

1. 玉ねぎを粗みじんに切る
2. 鶏モモは皮を外し一口大に切る
3. 鍋にオリーブオイルを熱したまねぎ、にんにく、しょうが、塩をいれ中火で炒める
4. たまねぎが半透明になったらトマト缶とカレースバイスを入れる
5. 強火にかけ全体を混ぜる
6. 鶏モモ肉、水100㎖、ケチャップ、ウスターソースを加え蓋をして中火で煮る
7. 鶏肉に火が通ったら蓋をとり好みのトロミ具合になるまで煮る

鶏モモでのレシピですが、鶏むね肉やサバ缶、ノンオイルツナ缶へ変えるとより脂質を減らせアレンジできます。
是非作ってみてくださいね。
```

まずは、必要なライブラリをダウンロードします。

```sh
pip install -U oci
```

次に、OCI Agents を操作するために必要なライブラリをインポートします。

```py
from oci.auth.signers import InstancePrincipalsSecurityTokenSigner
from oci.generative_ai_agent_runtime.generative_ai_agent_runtime_client import GenerativeAiAgentRuntimeClient
from oci.generative_ai_agent_runtime.models import ChatDetails, CreateSessionDetails
```

Agents から提供されている実行環境を使うためのクライアントを初期化します。

```py
signer = InstancePrincipalsSecurityTokenSigner()
agents_client = GenerativeAiAgentRuntimeClient(
    config={},
    signer=signer,
    service_endpoint="https://agent-runtime.generativeai.us-chicago-1.oci.oraclecloud.com" # Chicagoリージョンから先行提供されているので、エンドポイントは明示的に指定する
)
```

上記の `signer` は、OCI の Compute 上で実行することを前提としていますが、API Key(`~/.oci/config`) を使う場合は以下のようにすると良いでしょう。

```py
from oci.config import from_file
config = from_file()
agents_client = GenerativeAiAgentRuntimeClient(
    config=config,
    service_endpoint="https://agent-runtime.generativeai.us-chicago-1.oci.oraclecloud.com"
)
```

次にエージェントが会話全体で一貫した出力ができるようにセッションを作成します。

```py
import uuid

session_id = agents_client.create_session(
    create_session_details=CreateSessionDetails(
        display_name=str(uuid.uuid4()),
    ),
    agent_endpoint_id="ocid1.genaiagentendpoint.oc1.us-chicago-1.amaaaaaassl65iqanmqhxcqhsdlk25bhrk5jgarg3lq53o2wnt4y2xb6du2q"
).data.id
```

払い出したセッション ID を使い OCI Agents を実行します。

```py
response = agents_client.chat(
    agent_endpoint_id="ocid1.genaiagentendpoint.oc1.us-chicago-1.amaaaaaassl65iqanmqhxcqhsdlk25bhrk5jgarg3lq53o2wnt4y2xb6du2q",
    chat_details=ChatDetails(
        user_message="カレーの作り方を日本語で教えて！",
        session_id=session_id,
        should_stream=False
    )
)

print(response.data.message.content.text)
```

実行結果は、以下の通りです。

```txt
スパイスカレーの作り方は以下の通りです。

まずは、材料をご用意ください。材料は2人分です。

- 鶏モモ肉皮なし: 200g
- 玉ねぎ1/2: 110g
- にんにく: 4g（チューブ8cm）
- しょうが: 4g（チューブ8cm）
- 塩: 2g
- オリーブ油: 8g
- トマト缶1/3: 60g
- カレー粉: 5g
- ガラムマサラ: 1g
- ケチャップ: 18g
- ウスターソース: 18g

1. 玉ねぎを粗みじんに切ります。
2. 鶏モモは皮を取り除き、一口大に切ります。
3. 鍋にオリーブオイルを熱し、玉ねぎ、にんにく、しょうが、塩を入れて中火で炒めます。
4. 玉ねぎが半透明になったら、トマト缶とカレー粉を加えます。
5. 強火にし、全体を混ぜ合わせます。
6. 鶏モモ肉、水100ml、ケチャップ、ウスターソースを加え、蓋をして中火で煮ます。
7. 鶏肉に火が通ったら、蓋を取り、好みのとろみになるまで煮込みます。

鶏モモ肉の代わりに鶏むね肉やサバ缶、ノンオイルツナ缶を使用すると、脂質を減らすことができ、アレンジもききます。ぜひお試しください。
```

しっかりと、ドキュメントを参照して回答が作成されていることが確認できました。レスポンスには、回答の根拠を示す引用（`citations`）が含まれていたり、OCI Agents がどのような順番で最終的な出力をしたか（`trace`）が含まれています。

引用（`citations`）：

```py
print(response.data.content.citations)
```

実行結果

```py
[{
  "source_location": {
    "source_location_type": "OCI_OBJECT_STORAGE",
    "url": "https://objectstorage.us-chicago-1.oraclecloud.com/n/<namespace>/b/<bucket-name>/o/spice-curry.txt"
  },
  "source_text": "... 割愛 ..."
}, {
  "source_location": {
    "source_location_type": "OCI_OBJECT_STORAGE",
    "url": "https://objectstorage.us-chicago-1.oraclecloud.com/n/<namespace>/b/<bucket-name>/o/spice-curry.txt"
  },
  "source_text": "... 割愛 ..."
}]
```

トレース情報（`traces`）：

```py
print(response.data.traces)
```

実行結果

```py
[{
  "citations": [
    {
      "source_location": {
        "source_location_type": "OCI_OBJECT_STORAGE",
        "url": "https://objectstorage.us-chicago-1.oraclecloud.com/n/<namespace>/b/<bucket-name>/o/spice-curry.txt"
      },
      "source_text": "... 割愛 ..."
    },
    {
      "source_location": {
        "source_location_type": "OCI_OBJECT_STORAGE",
        "url": "https://objectstorage.us-chicago-1.oraclecloud.com/n/<namespace>/b/<bucket-name>/o/spice-curry.txt"
      },
      "source_text": "... 割愛 ..."
    }
  ],
  "retrieval_input": "... 割愛 ...",
  "time_created": "2024-10-02T07:29:06.344000+00:00",
  "trace_type": "RETRIEVAL_TRACE"
}, {
  "generation": "... 割愛 ...",
  "time_created": "2024-10-02T07:29:18.213000+00:00",
  "trace_type": "GENERATION_TRACE"
}]
```

これを確認すると、いつ検索が行われていつ LLM の生成タスクが行われたのか？といった情報を確認することが可能です。

# 終わりに

LLM を活用したアプリケーションを構築する前提で SDK を用いて、OCI Agents を使ってみました。スクラッチで作る場合に比べてだいぶ楽に実装できることが確認できたと思います。また、 ~~大分簡易的ですが~~トレース情報も提供されているので、内部でどのような処理が行われているのかも確認できますので、気になった方はぜひ触ってみてください！
