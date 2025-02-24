---
title: ベクトル検索評価用の大量データの埋め込み（Embeddings）を取得する
tags:
  - "oci"
private: false
updated_at: ""
id: null
organization_url_name: oracle
slide: false
ignorePublish: false
---

# はじめに

近年の LLM を活用したアプリケーションでは、モデルそのものが有している知識を補完するために何かしらの検索システムから得られた結果をプロンプトに差し込む構成（Retrieval-Augmented Generation）を取られるパターンが依然として多いと思います。この時、ユーザーの入力を受けてから出力を返すまでの一連の流れの中で最も時間がかかるのが LLM を介した処理となります。が、ここを改善するためには自身で LLM をホストしていない限りはできることは限られてくるでしょう。次に実行時間として支配的なのが検索システムに対する問い合わせとなります。つまり、機能・非機能要件に沿った検索システムを選定することが重要となります。本記事では、その検索システムにおいて広く使われているベクトルデータベースの性能（QPS など）を評価するためのデータセットを効率的に作成することを目的とします。

# 使うデータセット

- [singletongue/wikipedia-utils](https://huggingface.co/datasets/singletongue/wikipedia-utils)
  - 日本語版 Wikipedia のデータに対してチャンキングなどの前処理が行われたデータセットです
  - 今回は、`passages-c400-jawiki-20240401` という 400 文字ごとに分割されたデータセットを用いました

# 実行環境について

- データセットに対する埋め込みは、OCI Generative AI Service(cohere.embed-multilingual-v3.0)を使用します
- Python プログラムの実行環境は、4vCPU(2OCPU) 以上が搭載された環境であることを前提としています

# スクリプトについて

以下のようなスクリプトを作成してみました。

```bulk-embeddings.py
import multiprocessing as mp
import threading, csv, os, datetime, sys
from concurrent.futures import ThreadPoolExecutor, as_completed
from datasets import (
    load_dataset,
    DatasetDict,
    Dataset,
    IterableDataset,
    IterableDatasetDict,
)
from oci.retry import ExponentialBackoffWithFullJitterRetryStrategy
from oci.retry.retry_checkers import RetryCheckerContainer, LimitBasedRetryChecker
from oci.config import from_file
from oci.auth.signers import InstancePrincipalsSecurityTokenSigner
from oci.generative_ai_inference.generative_ai_inference_client import (
    GenerativeAiInferenceClient,
)
from oci.generative_ai_inference.models import EmbedTextDetails, OnDemandServingMode

BATCH_SIZE = 96
MAX_THREADS = min(mp.cpu_count(), 4)
OUTPUT_FILE = f"wiki_ja_embeddings_{datetime.date.today()}.csv"
CSV_COLUMNS = ["id", "pageid", "revid", "title", "section", "text", "embedding"]
LOCK = threading.Lock()

# 環境に応じて設定
COMPARTMENT_ID = os.getenv("COMPARTMENT_ID")
if COMPARTMENT_ID == None:
    print(f"compartment_idを設定してください")
    sys.exit(1)
REGION = os.getenv("REGION", "us-chicago-1")
USE_IP = os.getenv("USE_IP", False)

checker_container = RetryCheckerContainer(checkers=[LimitBasedRetryChecker()])
retry_strategy = ExponentialBackoffWithFullJitterRetryStrategy(
    base_sleep_time_seconds=10,
    exponent_growth_factor=2,
    max_wait_between_calls_seconds=30,
    checker_container=checker_container
)

if USE_IP == False:
    print(f"Use default config")
    generative_ai_client = GenerativeAiInferenceClient(
        config=from_file(),
        service_endpoint=f"https://inference.generativeai.{REGION}.oci.oraclecloud.com",
        retry_strategy=retry_strategy,
    )
else:
    print(f"Use Instance Principal")
    generative_ai_client = GenerativeAiInferenceClient(
        config={},
        signer=InstancePrincipalsSecurityTokenSigner(),
        service_endpoint=f"https://inference.generativeai.{REGION}.oci.oraclecloud.com",
        retry_strategy=retry_strategy,
    )

with open(OUTPUT_FILE, "w", newline="", encoding="utf-8") as f:
    writer = csv.writer(f)
    writer.writerow(CSV_COLUMNS)


def get_text_embeddings(texts: list[str]) -> list[list[float]]:
    try:
        response = generative_ai_client.embed_text(
            EmbedTextDetails(
                inputs=texts,
                serving_mode=OnDemandServingMode(
                    model_id="cohere.embed-multilingual-v3.0"
                ),
                compartment_id=COMPARTMENT_ID,
                input_type="SEARCH_DOCUMENT",
            )
        )
        return response.data.embeddings
    except Exception as e:
        print(f"Embeddingの取得に失敗しました: {e}")
        return [None] * len(texts)


def batch_processing(batch):
    ids = batch["id"]
    page_ids = batch["pageid"]
    revids = batch["revid"]
    titles = batch["title"]
    sections = batch["section"]
    texts = batch["text"]
    embeddings = get_text_embeddings(texts)
    # OOM(Out-of-Memory)が発生しないようにバッチ処置単位でロックをとりながらCSVに書き出し
    with LOCK:
        with open(OUTPUT_FILE, "a", newline="", encoding="utf-8") as f:
            writer = csv.writer(f)
            for i in range(len(batch)):
                writer.writerow(
                    [
                        ids[i],
                        page_ids[i],
                        revids[i],
                        titles[i],
                        sections[i],
                        texts[i],
                        embeddings[i],
                    ]
                )


def load_wikipedia_japanese_datasets() -> (
    DatasetDict | Dataset | IterableDatasetDict | IterableDataset
):
    # ローカルにデータセットが既に存在すればそれを再利用させる
    wiki_ja = load_dataset(
        "singletongue/wikipedia-utils",
        "passages-c400-jawiki-20240401",
        split="train",
        cache_dir="./datasets",
        trust_remote_code=True,
    )
    print(f"{wiki_ja=}")
    return wiki_ja


def main():
    wiki_ja = load_wikipedia_japanese_datasets()
    # 96個ずつのバッチに分割
    batches = [wiki_ja[i : i + BATCH_SIZE] for i in range(0, len(wiki_ja), BATCH_SIZE)]
    print(f"{len(batches)=}")
    with ThreadPoolExecutor(max_workers=MAX_THREADS) as executor:
        futures = {executor.submit(batch_processing, batch): batch for batch in batches}
        for future in as_completed(futures):
            try:
                future.result()
            except Exception as e:
                print(f"バッチ処理中にエラーが発生しました: {e}")


if __name__ == "__main__":
    main()
```

## ポイントの解説

### OCI Generative AI Service の呼び出しについて

https://docs.oracle.com/ja-jp/iaas/Content/generative-ai/use-playground-embed.htm

に記載されている通り、1 回の実行ごとに 96 の入力までサポートされるため、可能な限りまとめて処理させることがポイントです。Wikipedia くらいのデータ数で 1 回の実行で 1 つの入力しか与えないといつまで経っても完了しません。

```py
def get_text_embeddings(texts: list[str]) -> list[list[float]]:
    try:
        response = generative_ai_client.embed_text(
            EmbedTextDetails(
                inputs=texts,
                serving_mode=OnDemandServingMode(
                    model_id="cohere.embed-multilingual-v3.0"
                ),
                compartment_id=COMPARTMENT_ID,
                input_type="SEARCH_DOCUMENT",
            )
        )
        return response.data.embeddings
    # ...

def main():
    wiki_ja = load_wikipedia_japanese_datasets()
    # 96個ずつのバッチに分割
    batches = [wiki_ja[i : i + BATCH_SIZE] for i in range(0, len(wiki_ja), BATCH_SIZE)]
    # ...
```

これで 1 回の実行を効率化することができますが、後述する通りより現実的な時間で完了させるためにマルチスレッドで処理を行っています。多重度を上げると OCI Generative Service で 429 が返却され始めたので、それを適切に処理するために Exponential Backoff[^1] でリトライ処理を行います。幸いなことに OCI SDK にはこれを実現する組み込み実装が提供されていますのでそれを活用します。

```py
checker_container = RetryCheckerContainer(checkers=[LimitBasedRetryChecker()])
retry_strategy = ExponentialBackoffWithFullJitterRetryStrategy(
    base_sleep_time_seconds=10,
    exponent_growth_factor=2,
    max_wait_between_calls_seconds=30,
    checker_container=checker_container
)

generative_ai_client = GenerativeAiInferenceClient(
    config=from_file(),
    service_endpoint=f"https://inference.generativeai.{REGION}.oci.oraclecloud.com",
    retry_strategy=retry_strategy,
)
```

[^1]: リトライの待ち時間の間隔を指数関数的（Exponential）に増加させながら再試行するリトライ手法のこと

今回は、`ExponentialBackoffWithEqualJitterRetryStrategy` という指数関数的にリトライ間隔を増加させながら揺らぎ（Jitter）を加える実装を用いました。

### マルチスレッドで処理する

上記内容の並列度を上げることも重要です。今回は、実行環境の vCPU 数 or 4 のうち小さい方を採用しています。

```py
MAX_THREADS = min(mp.cpu_count(), 4)

# ...

def main():
    wiki_ja = load_wikipedia_japanese_datasets()
    # 96個ずつのバッチに分割
    batches = [wiki_ja[i : i + BATCH_SIZE] for i in range(0, len(wiki_ja), BATCH_SIZE)]
    print(f"{len(batches)=}")
    with ThreadPoolExecutor(max_workers=MAX_THREADS) as executor:
        futures = {executor.submit(batch_processing, batch): batch for batch in batches}
        for future in as_completed(futures):
            try:
                future.result()
            except Exception as e:
                print(f"バッチ処理中にエラーが発生しました: {e}")
```

スロットリングが発生した時のために、リトライ戦略は実装済みですが、念のため並列度を 4 程度に絞るようにしています。

### OOM(Out of Memory)とならないように定期的に CSV に出力する

今回の埋め込みは、1024 次元のため取得した埋め込みをメモリ上に保持しておくためには `1024 * 4 [byte] * 5,807,053 [rows] = 23.7 [GB]` 程度必要になるため定期的に CSV に出力させています。

```py
OUTPUT_FILE = f"wiki_ja_embeddings_{datetime.date.today()}.csv"
CSV_COLUMNS = ["id", "pageid", "revid", "title", "section", "text", "embedding"]
LOCK = threading.Lock()

# ...

def batch_processing(batch):
    ids = batch["id"]
    page_ids = batch["pageid"]
    revids = batch["revid"]
    titles = batch["title"]
    sections = batch["section"]
    texts = batch["text"]
    embeddings = get_text_embeddings(texts)
    # OOM(Out-of-Memory)が発生しないようにバッチ処置単位でロックをとりながらCSVに書き出し
    with LOCK:
        with open(OUTPUT_FILE, "a", newline="", encoding="utf-8") as f:
            writer = csv.writer(f)
            for i in range(len(batch)):
                writer.writerow(
                    [
                        ids[i],
                        page_ids[i],
                        revids[i],
                        titles[i],
                        sections[i],
                        texts[i],
                        embeddings[i],
                    ]
                )
```

# 終わりに

Wikipedia のような大規模なデータセットを用いて、ベクトル検索を評価するためのデータセット作成で使えそうなスクリプトを作ってみました。CSV として出力しているので、さまざまなデータベースにバルクでインサートできるはずです（多分）。
