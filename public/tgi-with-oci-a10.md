---
title: Hugging Face TGI(Text Generation Inference) を OCI の A10 で試してみた
tags:
  - "oci"
  - "huggingface"
private: false
updated_at: ""
id: null
organization_url_name: oracle
slide: false
ignorePublish: false
---

# はじめに

LLM(Large Language Model/大規模言語モデル)の推論環境を API として提供する場合、Fast API, Flask などで推論部分をラップして提供する方法の他に Hugging Face の TGI(Text Generation Inference)を使う方法があります。自前で実装する場合、細かく調整をすることができますが、その分複数のパラメータ（`max_new_tokens`, `top_k`, ...）をサポートする必要があったりと、実装はそれなりに手間がかかるものです。TGI では、主要なエンドポイントとパラメータがサポートされているので、ローカルで手軽に試す分には非常に体験が良いものだと思いました。そこで、今回はキャッチアップがてら Hugging Face TGI を OCI から提供されている A10 を使って試してみたいと思います。

# Hugging Face TGI

Hugging Face TGI は、Rust, Python で実装されている LLM の推論 API を提供するためのツールです。

![TGI.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/dc2a1291-a5cf-4a06-7326-03e9a051374d.png)

引用: [Hugging Face - Text Generation Inference](https://huggingface.co/docs/text-generation-inference/index)

図を見てもらうと分かる通り、Web Server 部分は Rust で実装されていて、その裏側の推論部分は Python で実装されており、それが gRPC 経由で呼び出されています。サポートされているエンドポイントは、[https://huggingface.github.io/text-generation-inference/](https://huggingface.github.io/text-generation-inference/) で公開されています。一見してみると、テキスト生成のための `/generate` やそれをストリーム出力するための `/generate_stream`、入力テキストをトークン化するための `/tokenize`、チャット用の `/v1/chat/completions`, `/v1/completions` などがサポートされています。

また、OpenTelemetry による分散トレーシングをサポートしていたり、OpenMetrics 形式でメトリクスを公開するエンドポイント（`/metrics`）を有していたりとオブザーバビリティに関する仕組みも十分に備わっているみたいです。

# OCI の A10 上で TGI を試してみる

## 前提

- OCI 上に A10(VM)が構築されていること
- LLM は任意ですが、今回は [Swallow-7b-hf](https://huggingface.co/tokyotech-llm/Swallow-7b-hf) を使用します
- Docker に関する基本的な操作ができること
- NVIDIA Container Toolkit がインストール済みであること
  - 未実施の方は、[https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) をご参照ください

## Swallow-7b-hf の推論環境を構築してみる

以下のように起動します。

```sh
docker container run --gpus all --shm-size 64g -p 8080:80 \
  -v $PWD/data:/data \
  ghcr.io/huggingface/text-generation-inference:2.0.3 \
  --model-id tokyotech-llm/Swallow-7b-hf
```

ボリュームマウントをしておくと、実行のたびにモデルのダウンロード処理が走ることがなくなるので、入れておくのが良いです。

実行すると、以下のようなログが出力されます。（初回実行時には、モデルのダウンロード処理が走るため少し時間を要します）

```log
2024-05-31T05:53:06.994911Z  INFO text_generation_launcher: Args {
    model_id: "tokyotech-llm/Swallow-7b-hf",
    revision: None,
    validation_workers: 2,
    sharded: None,
    num_shard: None,
    quantize: None,
    speculate: None,
    dtype: None,
    trust_remote_code: false,
    max_concurrent_requests: 128,
    max_best_of: 2,
    max_stop_sequences: 4,
    max_top_n_tokens: 5,
    max_input_tokens: None,
    max_input_length: None,
    max_total_tokens: None,
    waiting_served_ratio: 0.3,
    max_batch_prefill_tokens: None,
    max_batch_total_tokens: None,
    max_waiting_tokens: 20,
    max_batch_size: None,
    cuda_graphs: None,
    hostname: "4ffe1b16383f",
    port: 80,
    shard_uds_path: "/tmp/text-generation-server",
    master_addr: "localhost",
    master_port: 29500,
    huggingface_hub_cache: Some(
        "/data",
    ),
    weights_cache_override: None,
    disable_custom_kernels: false,
    cuda_memory_fraction: 1.0,
    rope_scaling: None,
    rope_factor: None,
    json_output: false,
    otlp_endpoint: None,
    cors_allow_origin: [],
    watermark_gamma: None,
    watermark_delta: None,
    ngrok: false,
    ngrok_authtoken: None,
    ngrok_edge: None,
    tokenizer_config_path: None,
    disable_grammar_support: false,
    env: false,
    max_client_batch_size: 4,
}
2024-05-31T05:53:06.994984Z  INFO hf_hub: Token file not found "/root/.cache/huggingface/token"
2024-05-31T05:53:07.377899Z  INFO text_generation_launcher: Default `max_input_tokens` to 4095
2024-05-31T05:53:07.377913Z  INFO text_generation_launcher: Default `max_total_tokens` to 4096
2024-05-31T05:53:07.377915Z  INFO text_generation_launcher: Default `max_batch_prefill_tokens` to 4145
2024-05-31T05:53:07.377917Z  INFO text_generation_launcher: Using default cuda graphs [1, 2, 4, 8, 16, 32]
2024-05-31T05:53:07.377987Z  INFO download: text_generation_launcher: Starting download process.
2024-05-31T05:53:11.016162Z  WARN text_generation_launcher: No safetensors weights found for model tokyotech-llm/Swallow-7b-hf at revision None. Downloading PyTorch weights.

2024-05-31T05:53:11.196666Z  INFO text_generation_launcher: Download file: pytorch_model-00001-of-00002.bin

2024-05-31T05:53:25.226283Z  INFO text_generation_launcher: Downloaded /data/models--tokyotech-llm--Swallow-7b-hf/snapshots/84f3f991fe44779f2995796ef6478c81d4456c9d/pytorch_model-00001-of-00002.bin in 0:00:14.

2024-05-31T05:53:25.226358Z  INFO text_generation_launcher: Download: [1/2] -- ETA: 0:00:14

2024-05-31T05:53:25.226592Z  INFO text_generation_launcher: Download file: pytorch_model-00002-of-00002.bin

2024-05-31T05:53:30.904069Z  INFO text_generation_launcher: Downloaded /data/models--tokyotech-llm--Swallow-7b-hf/snapshots/84f3f991fe44779f2995796ef6478c81d4456c9d/pytorch_model-00002-of-00002.bin in 0:00:05.

2024-05-31T05:53:30.904152Z  INFO text_generation_launcher: Download: [2/2] -- ETA: 0

2024-05-31T05:53:30.904227Z  WARN text_generation_launcher: 🚨🚨BREAKING CHANGE in 2.0🚨🚨: Safetensors conversion is disabled without `--trust-remote-code` because Pickle files are unsafe and can essentially contain remote code execution!Please check for more information here: https://huggingface.co/docs/text-generation-inference/basic_tutorials/safety

2024-05-31T05:53:30.904287Z  WARN text_generation_launcher: No safetensors weights found for model tokyotech-llm/Swallow-7b-hf at revision None. Converting PyTorch weights to safetensors.

2024-05-31T05:53:46.719463Z  INFO text_generation_launcher: Convert: [1/2] -- Took: 0:00:15.451689

2024-05-31T05:53:52.433858Z  INFO text_generation_launcher: Convert: [2/2] -- Took: 0:00:05.714075

2024-05-31T05:53:52.913530Z  INFO download: text_generation_launcher: Successfully downloaded weights.
2024-05-31T05:53:53.041688Z  INFO shard-manager: text_generation_launcher: Starting shard rank=0
2024-05-31T05:54:02.056947Z  INFO text_generation_launcher: Server started at unix:///tmp/text-generation-server-0

2024-05-31T05:54:02.150228Z  INFO shard-manager: text_generation_launcher: Shard ready in 9.107832275s rank=0
2024-05-31T05:54:02.220946Z  INFO text_generation_launcher: Starting Webserver
2024-05-31T05:54:02.242553Z  INFO text_generation_router: router/src/main.rs:195: Using the Hugging Face API
2024-05-31T05:54:02.242597Z  INFO hf_hub: /usr/local/cargo/registry/src/index.crates.io-6f17d22bba15001f/hf-hub-0.3.2/src/lib.rs:55: Token file not found "/root/.cache/huggingface/token"
2024-05-31T05:54:02.822118Z  INFO text_generation_router: router/src/main.rs:474: Serving revision 84f3f991fe44779f2995796ef6478c81d4456c9d of model tokyotech-llm/Swallow-7b-hf
2024-05-31T05:54:02.822185Z  INFO text_generation_router: router/src/main.rs:289: Using config Some(Llama)
2024-05-31T05:54:02.822197Z  WARN text_generation_router: router/src/main.rs:291: Could not find a fast tokenizer implementation for tokyotech-llm/Swallow-7b-hf
2024-05-31T05:54:02.822199Z  WARN text_generation_router: router/src/main.rs:292: Rust input length validation and truncation is disabled
2024-05-31T05:54:02.825450Z  INFO text_generation_router: router/src/main.rs:317: Warming up model
2024-05-31T05:54:04.410417Z  INFO text_generation_launcher: Cuda Graphs are enabled for sizes [1, 2, 4, 8, 16, 32]

2024-05-31T05:54:05.264187Z  INFO text_generation_router: router/src/main.rs:354: Setting max batch total tokens to 15232
2024-05-31T05:54:05.264204Z  INFO text_generation_router: router/src/main.rs:355: Connected
2024-05-31T05:54:05.264207Z  WARN text_generation_router: router/src/main.rs:369: Invalid hostname, defaulting to 0.0.0.0
```

## LLM の推論を実施してみる

### with curl

テキスト生成（同期）

```sh
curl http://localhost:8080/generate \
    -X POST \
    -d '{"inputs":"日本で一番高い山はなんですか？","parameters":{"max_new_tokens":200}}' \
    -H 'Content-Type: application/json'
```

実行結果:

```sh
{"generated_text":"\n日本で一番高い山はなんですか?\nベストアンサー\n富士山です。 日本で一番高い山は富士山です。 富士山は、静岡県と山梨県にまたがる標高3,776mの山です。 日本で一番高い山は富士山です。 富士山は、静岡県と山梨県にまたがる標高3,776mの山です。 富士山は、静岡県と山梨県にまたがる標高3,776mの山です。 富士山は、静岡県と山梨県にまたがる標高3,776mの山です。 富士山は、静岡県と山梨県にまたがる標高3,776mの山です。 富士山は、静岡県と山梨県にまたがる標高3,776mの山です。 富士山は、静岡県と山梨県にまたがる標高"}
```

テキスト生成（ストリーム出力）

```sh
curl http://localhost:8080/generate_stream \
    -X POST \
    -d '{"inputs":"日本で一番高い山はなんですか？","parameters":{"max_new_tokens":20}}' \
    -H 'Content-Type: application/json'
```

実行結果:

```sh
data:{"index":1,"token":{"id":13,"text":"\n","logprob":-0.5283203,"special":false},"generated_text":null,"details":null}

data:{"index":2,"token":{"id":32072,"text":"日本","logprob":-2.4394531,"special":false},"generated_text":null,"details":null}

data:{"index":3,"token":{"id":30499,"text":"で","logprob":-0.54785156,"special":false},"generated_text":null,"details":null}

data:{"index":4,"token":{"id":32611,"text":"一番","logprob":-0.114868164,"special":false},"generated_text":null,"details":null}

data:{"index":5,"token":{"id":32246,"text":"高い","logprob":-0.02911377,"special":false},"generated_text":null,"details":null}

data:{"index":6,"token":{"id":30329,"text":"山","logprob":-0.015052795,"special":false},"generated_text":null,"details":null}

data:{"index":7,"token":{"id":30449,"text":"は","logprob":-0.18432617,"special":false},"generated_text":null,"details":null}

data:{"index":8,"token":{"id":32126,"text":"なん","logprob":-1.1835938,"special":false},"generated_text":null,"details":null}

data:{"index":9,"token":{"id":32001,"text":"です","logprob":-0.012969971,"special":false},"generated_text":null,"details":null}

data:{"index":10,"token":{"id":30412,"text":"か","logprob":-0.0022029877,"special":false},"generated_text":null,"details":null}

data:{"index":11,"token":{"id":29973,"text":"?","logprob":-0.06573486,"special":false},"generated_text":null,"details":null}

data:{"index":12,"token":{"id":13,"text":"\n","logprob":-0.19824219,"special":false},"generated_text":null,"details":null}

data:{"index":13,"token":{"id":33676,"text":"ベスト","logprob":-1.2333984,"special":false},"generated_text":null,"details":null}

data:{"index":14,"token":{"id":32709,"text":"アン","logprob":-0.00019824505,"special":false},"generated_text":null,"details":null}

data:{"index":15,"token":{"id":32119,"text":"サー","logprob":-0.0000014305115,"special":false},"generated_text":null,"details":null}

data:{"index":16,"token":{"id":13,"text":"\n","logprob":-0.0010671616,"special":false},"generated_text":null,"details":null}

data:{"index":17,"token":{"id":35072,"text":"富士","logprob":-0.8286133,"special":false},"generated_text":null,"details":null}

data:{"index":18,"token":{"id":30329,"text":"山","logprob":-0.003786087,"special":false},"generated_text":null,"details":null}

data:{"index":19,"token":{"id":32001,"text":"です","logprob":-0.58496094,"special":false},"generated_text":null,"details":null}

data:{"index":20,"token":{"id":30267,"text":"。","logprob":-0.34350586,"special":false},"generated_text":"\n日本で一番高い山はなんですか?\nベストアンサー\n富士山です。","details":null}
```

### with Inference Client

Python のプログラム上から実行する場合は、[huggingface-hub](https://huggingface.co/docs/huggingface_hub/main/en/index) からクライアントライブラリが提供されているのでそれを使うのが良いでしょう。

```sh
pip install huggingface-hub
```

テキスト生成

```py
from huggingface_hub import InferenceClient

client = InferenceClient(model="http://localhost:8080")
response = client.text_generation(prompt="日本で一番高い山はなんですか？", max_new_tokens=200)
print(response)
```

実行結果:

```py
日本で一番高い山はなんですか?
ベストアンサー
富士山です。 日本で一番高い山は富士山です。 富士山は、静岡県と山梨県にまたがる標高3,776mの山です。 日本で一番高い山は富士山です。 富士山は、静岡県と山梨県にまたがる標高3,776mの山です。 富士山は、静岡県と山梨県にまたがる標高3,776mの山です。 富士山は、静岡県と山梨県にまたがる標高3,776mの山です。 富士山は、静岡県と山梨県にまたがる標高3,776mの山です。 富士山は、静岡県と山梨県にまたがる標高3,776mの山です。 富士山は、静岡県と山梨県にまたがる標高
```

テキスト生成（ストリーム出力）

```py
from huggingface_hub import InferenceClient

client = InferenceClient(model="http://localhost:8080")
response = client.text_generation(prompt="日本で一番高い山はなんですか？", max_new_tokens=20, stream=True)
for token in response:
    print(token, end='')
```

実行結果:

```py
日本で一番高い山はなんですか?
ベストアンサー
富士山です。
```

# 終わりに

今回は、手元で LLM の推論環境を簡単に提供する方法の 1 つとして Hugging Face TGI を試してみました。LLM に対する主要な操作やパラメータがサポートされており、クライアントライブラリまで提供されているので、非常に便利に使えそうだなと感じました。一方で、素敵だな！と思った OpenTelemetry を用いた分散トレーシングについては確認した限り、ドキュメントが見当たらずどのように計装するのか分かりませんでした。この辺りは、もう少し調査を進めてみたいと思います。

# 参考

https://huggingface.co/docs/text-generation-inference/index
