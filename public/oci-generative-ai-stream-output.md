---
title: OCI Generative AI Service の Stream 出力を REST API から実施する
tags:
  - oci
private: false
updated_at: '2024-02-19T17:35:24+09:00'
id: 2eef36158ef8088c72d3
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

OCI Generative AI Service が GA となり、おおよその機能は SDK から使うことができるようになっています。チャット等のインターフェースなどからこれを扱う場合には、ユーザー体験を悪くしないために生成されたテキストを順次クライアントに返却し、その内容をレンダリングするような工夫を一般的にはすると思いますが、SDK では現時点（※2024/02/19）でサポートされていません。しかし、REST API ではサポートされているようなので今回はこれを簡単に試してみます。

# 実装例

```py
import requests
import json
from oci.auth.signers import InstancePrincipalsSecurityTokenSigner

compartment_id = "<your-compartment-id>"
model_ocid = "<model-ocid>"
api_endpoint = "https://inference.generativeai.us-chicago-1.oci.oraclecloud.com"

signer = InstancePrincipalsSecurityTokenSigner()

prompt = "Tell me a joke."
body = {
    "compartmentId": compartment_id,
    "inferenceRequest": {
        "runtimeType": "COHERE",
        "isStream": True,
        "maxTokens": 500,
        "numGenerations": 1,
        "prompt": prompt
    },
    "servingMode": {
        "servingType": "ON_DEMAND",
        "modelId": model_ocid
    }
}

response = requests.post(
    url=api_endpoint + "/20231130/actions/generateText",
    json=body,
    auth=signer,
    stream=True
)

for chunk in response.iter_lines():
    if chunk.strip():
        content = json.loads(chunk.decode('utf-8').split(': ', 1)[1])
        print(content)
```

`inferenceRequest.isStream` に `True` を設定することと、`requests.post` で `stream=True` にするのがポイントです。上記のコードを実行すると、以下のように結果を得ることができます。

```sh
{'id': 'aaa6fc53-c156-4491-bccf-3629e26f6e5b', 'text': ' Why'}
{'id': 'aaa6fc53-c156-4491-bccf-3629e26f6e5b', 'text': ' was'}
{'id': 'aaa6fc53-c156-4491-bccf-3629e26f6e5b', 'text': ' the'}
{'id': 'aaa6fc53-c156-4491-bccf-3629e26f6e5b', 'text': ' computer'}
{'id': 'aaa6fc53-c156-4491-bccf-3629e26f6e5b', 'text': ' cold'}
{'id': 'aaa6fc53-c156-4491-bccf-3629e26f6e5b', 'text': '?'}
{'id': 'aaa6fc53-c156-4491-bccf-3629e26f6e5b', 'text': '\n'}
{'id': 'aaa6fc53-c156-4491-bccf-3629e26f6e5b', 'text': '\n'}
{'id': 'aaa6fc53-c156-4491-bccf-3629e26f6e5b', 'text': 'Because'}
{'id': 'aaa6fc53-c156-4491-bccf-3629e26f6e5b', 'text': ' it'}
{'id': 'aaa6fc53-c156-4491-bccf-3629e26f6e5b', 'text': ' left'}
{'id': 'aaa6fc53-c156-4491-bccf-3629e26f6e5b', 'text': ' its'}
{'id': 'aaa6fc53-c156-4491-bccf-3629e26f6e5b', 'text': ' Windows'}
{'id': 'aaa6fc53-c156-4491-bccf-3629e26f6e5b', 'text': ' open'}
# ...
```

後は、使う側で煮るなり焼くなりできるはずです！

## e.g. Jupyter Notebook 上で Stream 出力する

```py
for chunk in response.iter_lines():
    if chunk.strip():
        content = json.loads(chunk.decode('utf-8').split(': ', 1)[1])
        print(content.get('text', ''), end='')
```

# おわりに

SDK が Stream 出力に対応するまでの暫定対処ですが、ご参考までに。

# 参考

https://docs.oracle.com/en-us/iaas/api/#/en/generative-ai-inference/20231130/GenerateTextResult/GenerateText
