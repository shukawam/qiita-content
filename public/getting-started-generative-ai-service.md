---
title: OCI Generative AI Service を CLI & SDK(Python) から操作してみた
tags:
  - oci
  - GenerativeAI
  - Cohere
private: false
updated_at: '2024-01-31T09:23:05+09:00'
id: 067850711cd0d008bbbd
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

以前、OCI Generative AI Service が出る前に、その元となっている Cohere について最低限理解しておこう！という意図でこのような記事を書きました。

https://qiita.com/shukawam/items/a8febfce15d92a0131b1

そして、OCI Generative AI Service が GA になったため、現時点でどのようなことができるのか？を改めてキャッチアップしておこうと思い、本記事の執筆に至りました。

https://docs.oracle.com/en-us/iaas/releasenotes/changes/e7a3be2e-67c6-4619-938b-a74f257253bf/

:::note warn
本記事は、2024/01/24 時点の情報に基づいて書かれています。各インタフェースから使用可能な機能などについては随時更新されると思われるため、最新情報はドキュメント等の一次情報を参考にしてください。
:::

# ライブラリ等の前提

- Python: 3.11.7
- OCI CLI: 3.37.5
- OCI SDK for Python: 2.119.1
- Region: us-chicago-1
  - ※上記リージョンのみ GA のため試す際は、シカゴリージョンでお試しください

また、本記事では専用クラスタ（dedicated-cluster）に関することは、ほとんど触れないのでご了承ください。

## OCI Generative AI Service を OCI CLI から操作する

便宜上、Compartment OCID や各エンドポイントは、以下のように環境変数に設定しておきます。

```sh
export C=<your-compartment-ocid>
export GEN_AI_ENDPOINT=https://generativeai.us-chicago-1.oci.oraclecloud.com
export GEN_AI_INFERENCE_ENDPOINT=https://inference.generativeai.us-chicago-1.oci.oraclecloud.com
```

CLI のドキュメントを眺めてみると、管理用のコマンドである `generative-ai` と推論用のコマンドである `generative-ai-inference` が存在することが分かります。

まずは、管理用コマンドである `generative-ai` から見ていきます。（※`--help` から一部抜粋）

```sh
AVAILABLE COMMANDS
    • dedicated-ai-cluster # 専用クラスタの管理コマンド
        • change-compartment
        • create
        • delete
        • get
        • update
    • dedicated-ai-cluster-collection # 専用クラスタをリストするためのコマンド
        • list-dedicated-ai-clusters
    • endpoint # 専用クラスタのエンドポイント管理用コマンド
        • change-compartment
        • create
        • delete
        • get
        • update
    • endpoint-collection # エンドポイントをリストするためのコマンド
        • list-endpoints
    • model # モデル管理用のコマンド
        • change-compartment
        • create
        • delete
        • get
        • update
    • model-collection # モデルをリストするためのコマンド
        • list-models
    • work-request # 作業リクエスト管理用のコマンド（主に、専用クラスタ関連ジョブ管理に使用する）
        • get
        • list
    • work-request-error # 失敗した作業リクエストをリストするためのコマンド（主に、専用クラスタ関連に使用する）
        • list
    • work-request-log-entry # 作業リクエストのログエントリ関連コマンド（主に、専用クラスタ関連に使用する）
        • list-work-request-logs
```

ここでは、`model-collection` を試してみます。

```sh
oci generative-ai model-collection list-models \
    --endpoint $GEN_AI_ENDPOINT \
    --compartment-id $C
```

実行すると、現時点で使用可能なモデルとそのモデルがサポートしている機能やモデルの ID といった情報が返却されます。この ID は、後ほど推論時にモデルを指定するために使用します。

```json
{
  "data": {
    "items": [
      {
        "base-model-id": null,
        "capabilities": ["TEXT_GENERATION"],
        "compartment-id": null,
        "defined-tags": {},
        "display-name": "meta.llama-2-70b-chat",
        "fine-tune-details": null,
        "freeform-tags": {},
        "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyai3pxxkeezogygojnayizqu3bgslgcn6yiqvmyu3w75ma",
        "is-long-term-supported": null,
        "lifecycle-details": "Creating Base Model",
        "lifecycle-state": "ACTIVE",
        "model-metrics": null,
        "system-tags": {},
        "time-created": "2024-01-05T02:19:51.103000+00:00",
        "time-deprecated": null,
        "type": "BASE",
        "vendor": "meta",
        "version": "1.0"
      },
      {
        "base-model-id": null,
        "capabilities": ["TEXT_GENERATION", "FINE_TUNE"],
        "compartment-id": null,
        "defined-tags": {},
        "display-name": "cohere.command",
        "fine-tune-details": null,
        "freeform-tags": {},
        "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya4bk5foxjdxe6332b6dmkzrdtqdulyodbek7ewzt5gqnq",
        "is-long-term-supported": null,
        "lifecycle-details": "BaseModel created",
        "lifecycle-state": "ACTIVE",
        "model-metrics": null,
        "system-tags": {},
        "time-created": "2024-01-02T21:06:03.031000+00:00",
        "time-deprecated": null,
        "type": "BASE",
        "vendor": "cohere",
        "version": "15.6"
      },
      {
        "base-model-id": null,
        "capabilities": ["TEXT_GENERATION", "FINE_TUNE"],
        "compartment-id": null,
        "defined-tags": {},
        "display-name": "cohere.command-light",
        "fine-tune-details": null,
        "freeform-tags": {},
        "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyacw6jlwbfskxv5e4yqd5yupgaxcn6x4q3esrv7punjo5a",
        "is-long-term-supported": null,
        "lifecycle-details": "BaseModel created",
        "lifecycle-state": "ACTIVE",
        "model-metrics": null,
        "system-tags": {},
        "time-created": "2024-01-02T20:50:08.484000+00:00",
        "time-deprecated": null,
        "type": "BASE",
        "vendor": "cohere",
        "version": "15.6"
      },
      {
        "base-model-id": null,
        "capabilities": ["TEXT_SUMMARIZATION"],
        "compartment-id": null,
        "defined-tags": {},
        "display-name": "cohere.command",
        "fine-tune-details": null,
        "freeform-tags": {},
        "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya3yvtzkwd7y4lq6cl3mb22obgesqh2k4k7t2ruzix55ia",
        "is-long-term-supported": null,
        "lifecycle-details": "Base Model created",
        "lifecycle-state": "ACTIVE",
        "system-tags": {},
        "time-created": "2023-12-01T20:48:50.329000+00:00",
        "time-deprecated": null,
        "type": "BASE",
        "vendor": "cohere",
        "version": "15.6"
      },
      {
        "base-model-id": null,
        "capabilities": ["TEXT_GENERATION"],
        "compartment-id": null,
        "defined-tags": {},
        "display-name": "cohere.command",
        "fine-tune-details": null,
        "freeform-tags": {},
        "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyafhwal37hxwylnpbcncidimbwteff4xha77n5xz4m7p6a",
        "is-long-term-supported": null,
        "lifecycle-details": "Base Model created",
        "lifecycle-state": "ACTIVE",
        "model-metrics": null,
        "system-tags": {},
        "time-created": "2023-12-01T20:46:38.865000+00:00",
        "time-deprecated": null,
        "type": "BASE",
        "vendor": "cohere",
        "version": "15.6"
      },
      {
        "base-model-id": null,
        "capabilities": ["TEXT_GENERATION"],
        "compartment-id": null,
        "defined-tags": {},
        "display-name": "cohere.command-light",
        "fine-tune-details": null,
        "freeform-tags": {},
        "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyad4uybcjtw475o4uqja3oz3kzrfhln7joysjxj3npp2la",
        "is-long-term-supported": null,
        "lifecycle-details": "Base Model created",
        "lifecycle-state": "ACTIVE",
        "model-metrics": null,
        "system-tags": {},
        "time-created": "2023-12-01T20:42:45.284000+00:00",
        "time-deprecated": null,
        "type": "BASE",
        "vendor": "cohere",
        "version": "15.6"
      },
      {
        "base-model-id": null,
        "capabilities": ["TEXT_EMBEDDINGS"],
        "compartment-id": null,
        "defined-tags": {},
        "display-name": "cohere.embed-english-light-v3.0",
        "fine-tune-details": null,
        "freeform-tags": {},
        "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya6fw6mqgi2mcmehwvd7h2ysqe25rbhprgclypiyh46wra",
        "is-long-term-supported": null,
        "lifecycle-details": "Base Model created",
        "lifecycle-state": "ACTIVE",
        "model-metrics": null,
        "system-tags": {},
        "time-created": "2023-11-30T20:55:12.928000+00:00",
        "time-deprecated": null,
        "type": "BASE",
        "vendor": "cohere",
        "version": "3.0"
      },
      {
        "base-model-id": null,
        "capabilities": ["TEXT_EMBEDDINGS"],
        "compartment-id": null,
        "defined-tags": {},
        "display-name": "cohere.embed-english-v3.0",
        "fine-tune-details": null,
        "freeform-tags": {},
        "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya3bqursz5i2eeg5eesvnlrqj4mrdmi3infd4ve3kaqjva",
        "is-long-term-supported": null,
        "lifecycle-details": "Base Model created",
        "lifecycle-state": "ACTIVE",
        "model-metrics": null,
        "system-tags": {},
        "time-created": "2023-11-30T20:54:14.931000+00:00",
        "time-deprecated": null,
        "type": "BASE",
        "vendor": "cohere",
        "version": "3.0"
      },
      {
        "base-model-id": null,
        "capabilities": ["TEXT_EMBEDDINGS"],
        "compartment-id": null,
        "defined-tags": {},
        "display-name": "cohere.embed-multilingual-light-v3.0",
        "fine-tune-details": null,
        "freeform-tags": {},
        "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyaya37tnqbhe7fdikai6r2z5qfb3aq6mdg3z3kbeenmlha",
        "is-long-term-supported": null,
        "lifecycle-details": "Base Model created",
        "lifecycle-state": "ACTIVE",
        "model-metrics": null,
        "system-tags": {},
        "time-created": "2023-11-30T20:52:21.320000+00:00",
        "time-deprecated": null,
        "type": "BASE",
        "vendor": "cohere",
        "version": "3.0"
      },
      {
        "base-model-id": null,
        "capabilities": ["TEXT_EMBEDDINGS"],
        "compartment-id": null,
        "defined-tags": {},
        "display-name": "cohere.embed-multilingual-v3.0",
        "fine-tune-details": null,
        "freeform-tags": {},
        "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyadcwe6y5jt655ox5hkm76vq2wz3hs7ljbfhljvithlnoq",
        "is-long-term-supported": null,
        "lifecycle-details": "Base Model created",
        "lifecycle-state": "ACTIVE",
        "model-metrics": null,
        "system-tags": {},
        "time-created": "2023-11-30T20:51:06.303000+00:00",
        "time-deprecated": null,
        "type": "BASE",
        "vendor": "cohere",
        "version": "3.0"
      },
      {
        "base-model-id": null,
        "capabilities": ["TEXT_GENERATION", "FINE_TUNE"],
        "compartment-id": null,
        "defined-tags": {},
        "display-name": "cohere.command",
        "fine-tune-details": null,
        "freeform-tags": {},
        "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaapi24rzaaxh5ulr23qhwme4bbix3bu3x5opqswkaxpuirgcmo4m7a",
        "is-long-term-supported": null,
        "lifecycle-details": "Created BaseModel",
        "lifecycle-state": "ACTIVE",
        "model-metrics": null,
        "system-tags": {},
        "time-created": "2023-10-13T21:21:33.212000+00:00",
        "time-deprecated": null,
        "type": "BASE",
        "vendor": "cohere",
        "version": "14.2"
      },
      {
        "base-model-id": null,
        "capabilities": ["TEXT_GENERATION", "FINE_TUNE"],
        "compartment-id": null,
        "defined-tags": {},
        "display-name": "cohere.command-light",
        "fine-tune-details": null,
        "freeform-tags": {},
        "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaapi24rzaalcbnpexqvkxpwfdcqskcupap27qtvr7ew5ierjpbih2a",
        "is-long-term-supported": null,
        "lifecycle-details": "Created",
        "lifecycle-state": "ACTIVE",
        "model-metrics": null,
        "system-tags": {},
        "time-created": "2023-09-08T22:20:13.843000+00:00",
        "time-deprecated": null,
        "type": "BASE",
        "vendor": "cohere",
        "version": "14.2"
      },
      {
        "base-model-id": null,
        "capabilities": ["TEXT_EMBEDDINGS"],
        "compartment-id": null,
        "defined-tags": {},
        "display-name": "cohere.embed-english-light-v2.0",
        "fine-tune-details": null,
        "freeform-tags": {},
        "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaapi24rzaauhcci2xuk7263dbon4xnkjku5ifv6i45nzbz6rrn4joa",
        "is-long-term-supported": null,
        "lifecycle-details": "Created",
        "lifecycle-state": "ACTIVE",
        "model-metrics": null,
        "system-tags": {},
        "time-created": "2023-09-08T18:58:28.048000+00:00",
        "time-deprecated": null,
        "type": "BASE",
        "vendor": "cohere",
        "version": "2.0"
      }
    ]
  }
}
```

続いて、推論用のコマンドである `generative-ai-inference` についてです。（※`--help` から一部抜粋）
テキストの生成は、まだ CLI では対応していないようです。（今後のアップデートに期待）

```sh
AVAILABLE COMMANDS
    • embed-text-result # Embedding 用コマンド
        • embed-text
        • embed-text-dedicated-serving-mode
        • embed-text-on-demand-serving-mode
    • summarize-text-result # Summarize 用コマンド
        • summarize-text
        • summarize-text-dedicated-serving-mode
        • summarize-text-on-demand-serving-mode
```

順番に試していきます。

### `generative-ai-inference　embed-text-result embed-text`

まずは実行に必要なパラメータを確認します。

```sh
oci generative-ai-inference embed-text-result embed-text \
    --generate-full-command-json-input > embed-text.json
```

実行結果

```json
{
  "compartmentId": "string",
  "inputType": "SEARCH_DOCUMENT|SEARCH_QUERY|CLASSIFICATION|CLUSTERING",
  "inputs": ["string", "string"],
  "isEcho": true,
  "servingMode": [
    "This parameter should actually be a JSON object rather than an array - pick one of the following object variants to use",
    {
      "endpointId": "string",
      "servingType": "DEDICATED"
    },
    {
      "modelId": "string",
      "servingType": "ON_DEMAND"
    }
  ],
  "truncate": "NONE|START|END"
}
```

各パラメータに関して簡単に補足します。(\*: Required)

| parameters      | description                                                                                                                                                                         |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| \*compartmentId | Generative AI Service の実行が許可されたコンパートメントの OCID                                                                                                                     |
| inputType       | 入力となるタイプを選択します（SEARCH_DOCUMENT or SEARCH_QUERY or CLASSIFICATION or CLUSTERING）                                                                                     |
| \*inputs        | Embedding を行う文字列。各入力は 512 トークン未満である必要があり、実行ごとに最大で 96 の入力がサポートされる                                                                       |
| isEcho          | 元の入力を回答に含めるかどうか                                                                                                                                                      |
| \*servingMode   | モデルの提供方法（ON_DEMAND or DEDICATED）と使用するモデル、エンドポイントを指定する                                                                                                |
| truncate        | 最大トークン長を超えた入力に関する扱いを指定する（NONE: 入力が最大入力トークン長（512）以上となると、エラーを発生させる, START: 入力の最初から切り捨て, END: 入力の最後から切り捨て |

実行例:

```embed-text.json
{
  "compartmentId": "ocid1.compartment.oc1...",
  "inputs": [
    "Learn about the Employee Stock Purchase Plan",
    "Reassign timecard approvals during leave",
    "View my payslip online",
    "Learn about the Employee Stock Purchase Plan",
    "Reassign timecard approvals during leave",
    "View my payslip online",
    "Enroll in benefits",
    "Change my direct deposit",
    "Have my employment/income verified",
    "Request A Workplace Accommodation",
    "Submit my time card",
    "Report Information Security Incidents",
    "Review the Code of Conduct",
    "Review the Social Media Policy",
    "Review Corporate Information Security Policies",
    "Understand Compliance and Ethics",
    "Understand the Fiscal Year Calendar",
    "Change my email address",
    "Change my personal information",
    "Learn about Analyst Relations",
    "Learn about Business Skills",
    "Learn about Career Development",
    "Learn about Employee Resource Groups",
    "Learn about Information Security",
    "Learn about Leadership skills",
    "Learn about sustainability",
    "Learn about Technical skills",
    "Request a Phone Number",
    "Learn about video conferencing",
    "Learn about Organizational Distribution Lists",
    "Subscribe to Group Mailing Lists",
    "Find a Campus Map",
    "Obtain a security badge",
    "Obtain an office workspace",
    "Submit an ergonomics request",
    "Use printers",
    "Delegate workflows or transaction approvals while out on leave",
    "Tips for working remotely",
    "Volunteer",
    "Reassign workflow approvals while on vacation",
    "How to delegate timecard approvals",
    "Apply for Corporate Credit Card",
    "Book travel",
    "Submit an expense report"
  ],
  "isEcho": true,
  "servingMode": {
    "modelId": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya3bqursz5i2eeg5eesvnlrqj4mrdmi3infd4ve3kaqjva",
    "servingType": "ON_DEMAND"
  },
  "truncate": "NONE"
}
```

```sh
oci generative-ai-inference embed-text-result embed-text \
    --from-json file://embed-text-result.json \
    --endpoint $GEN_AI_INFERENCE_ENDPOINT
```

実行結果:

とてつもなく長いので、割愛します。ご自身でお試しください！

### `generative-ai-inference　embed-text-result embed-text-on-demand-serving-mode`, `generative-ai-inference embed-text-result embed-text-dedicated-serving-mode`

[generative-ai-inference embed-text-result embed-text](#generative-ai-inference-embed-text-result-embed-text) と内容が重複するため割愛します。

### `generative-ai-inference summarize-text-result summarize-text`

まずは実行に必要なパラメータを確認します。

```sh
oci generative-ai-inference summarize-text-result summarize-text \
    --generate-full-command-json-input > summarize-text.json
```

実行結果

```json
{
  "additionalCommand": "string",
  "compartmentId": "string",
  "extractiveness": "LOW|MEDIUM|HIGH|AUTO",
  "format": "PARAGRAPH|BULLETS|AUTO",
  "input": "string",
  "isEcho": true,
  "length": "SHORT|MEDIUM|LONG|AUTO",
  "servingMode": [
    "This parameter should actually be a JSON object rather than an array - pick one of the following object variants to use",
    {
      "endpointId": "string",
      "servingType": "DEDICATED"
    },
    {
      "modelId": "string",
      "servingType": "ON_DEMAND"
    }
  ],
  "temperature": "string"
}
```

各パラメータに関して簡単に補足します。(\*: Required)

| parameters        | description                                                                          |
| ----------------- | ------------------------------------------------------------------------------------ |
| additionalCommand | 要約の生成方法を変更するための自由形式の指示                                         |
| \*compartmentId   | Generative AI Service の実行が許可されたコンパートメントの OCID                      |
| extractiveness    | 要約を生成する際にどれくらい原文から抽出するか（LOW or MEDIUM or HIGH or AUTO）      |
| format            | 要約の出力形式を指定（PARAGRAPH or BULLETS or AUTO）                                 |
| \*input           | 要約対象となる原文（入力の最大トークンは、4,000）                                    |
| isEcho            | 元の入力を回答に含めるかどうか                                                       |
| length            | 生成される要約の長さを指定（SHORT or MEDIUM or LONG or AUTO）                        |
| \*servingMode     | モデルの提供方法（ON_DEMAND or DEDICATED）と使用するモデル、エンドポイントを指定する |
| temperature       | 生成される出力テキストのランダム性を指定(0~5)                                        |

実行例:

```summarize-text.json
{
  "compartmentId": "ocid1.compartment.oc1...",
  "extractiveness": "MEDIUM",
  "format": "BULLETS",
  "input": "Hi Team,\n I am so proud to be a part of such an incredible team that has just shipped a new cloud service. This service is going to have a huge impact on the productivity of our customers and it is all thanks to the hard work and dedication of each and every one of you.\n I know that this project has been a long and challenging one, but you have all risen to the challenge and have delivered an amazing product. Your attention to detail, your commitment to quality, and your determination to get this service out to our customers has been truly inspiring.\n Thank you for your hard work and for your dedication to making our company the best it can be. I am so proud to be a part of such a great team and I look forward to seeing what we can achieve together in the future.\n Congratulations!\n Thanks,\n Kathy",
  "isEcho": true,
  "length": "SHORT",
  "servingMode": {
    "modelId": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya3yvtzkwd7y4lq6cl3mb22obgesqh2k4k7t2ruzix55ia",
    "servingType": "ON_DEMAND"
  },
  "temperature": "1.0"
}
```

```sh
oci generative-ai-inference summarize-text-result summarize-text \
  --from-json file://summarize-text.json \
  --endpoint $GEN_AI_INFERENCE_ENDPOINT
```

実行結果:

```json
{
  "data": {
    "id": "67f354c8-564a-4ba6-9ecd-0ccd54ee9e2f",
    "input": "Hi Team,\n I am so proud to be a part of such an incredible team that has just shipped a new cloud service. This service is going to have a huge impact on the productivity of our customers and it is all thanks to the hard work and dedication of each and every one of you.\n I know that this project has been a long and challenging one, but you have all risen to the challenge and have delivered an amazing product. Your attention to detail, your commitment to quality, and your determination to get this service out to our customers has been truly inspiring.\n Thank you for your hard work and for your dedication to making our company the best it can be. I am so proud to be a part of such a great team and I look forward to seeing what we can achieve together in the future.\n Congratulations!\n Thanks,\n Kathy",
    "model-id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya3yvtzkwd7y4lq6cl3mb22obgesqh2k4k7t2ruzix55ia",
    "model-version": "15.6",
    "summary": "- Kathy expresses pride in the team for shipping a new cloud service that will greatly enhance the productivity of their customers.\n- She acknowledges the team's hard work, dedication and attention to detail in creating the service.\n- Kathy looks forward to more successes together in the future."
  }
}
```

### `generative-ai-inference　summarize-text-result summarize-text-on-demand-serving-mode`, `generative-ai-inference summarize-text-result summarize-text-dedicated-serving-mode`

[generative-ai-inference summarize-text-result summarize-text](#generative-ai-inference-summarize-text-result-summarize-text) と内容が重複するため割愛します。

## OCI Generative AI Service を OCI SDK for Python から操作する

SDK のドキュメントを眺めてみると、管理用クライアントである GenerativeAiClient と推論用クライアントである GenerativeAiInferenceClient が存在することが分かります。まずは、管理用クライアントである GenerativeAiClient を試してみます。

### GenerativeAiClient

メソッドとしては、以下が提供されています。

| **init**(config, \*\*kwargs)                        | Creates a new service client                                                                                  |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| change_dedicated_ai_cluster_compartment(…)          | Moves a dedicated AI cluster into a different compartment within the same tenancy.                            |
| change_endpoint_compartment(endpoint_id, …)         | Moves an endpoint into a different compartment within the same tenancy.                                       |
| change_model_compartment(model_id, …)               | Moves a custom model into a different compartment.                                                            |
| create_dedicated_ai_cluster(…)                      | Creates a dedicated AI cluster.                                                                               |
| create_endpoint(create_endpoint_details, …)         | Creates an endpoint.                                                                                          |
| create_model(create_model_details, \*\*kwargs)      | Creates a custom model by fine-tuning a base model with your own dataset.                                     |
| delete_dedicated_ai_cluster(…)                      | Deletes a dedicated AI cluster.                                                                               |
| delete_endpoint(endpoint_id, \*\*kwargs)            | Deletes an endpoint.                                                                                          |
| delete_model(model_id, \*\*kwargs)                  | Deletes a custom model.                                                                                       |
| get_dedicated_ai_cluster(…)                         | Gets information about a dedicated AI cluster.                                                                |
| get_endpoint(endpoint_id, \*\*kwargs)               | Gets information about an endpoint.                                                                           |
| get_model(model_id, \*\*kwargs)                     | Gets information about a custom model.                                                                        |
| get_work_request(work_request_id, \*\*kwargs)       | Gets the details of a work request.                                                                           |
| list_dedicated_ai_clusters(compartment_id, …)       | Lists the dedicated AI clusters in a specific compartment.                                                    |
| list_endpoints(compartment_id, \*\*kwargs)          | Lists the endpoints of a specific compartment.                                                                |
| list_models(compartment_id, \*\*kwargs)             | Lists the models in a specific compartment.                                                                   |
| list_work_request_errors(work_request_id, …)        | Lists the errors for a work request.                                                                          |
| list_work_request_logs(work_request_id, \*\*kwargs) | Lists the logs for a work request.                                                                            |
| list_work_requests(compartment_id, \*\*kwargs)      | Lists the work requests in a compartment.                                                                     |
| update_dedicated_ai_cluster(…)                      | Updates a dedicated AI cluster.                                                                               |
| update_endpoint(endpoint_id, …)                     | Updates the properties of an endpoint.                                                                        |
| update_model(model_id, update_model_details, …)     | Updates the properties of a custom model such as name, description, version, freeform tags, and defined tags. |

※[oci.generative_ai.GenerativeAiClient](https://docs.oracle.com/en-us/iaas/tools/python/2.119.1/api/generative_ai/client/oci.generative_ai.GenerativeAiClient.html#oci.generative_ai.GenerativeAiClient) から引用

ここでは、CLI と同様にモデルの一覧をリストする `list_models(compartment_id, **kwargs)` をみていきます。尚、以降のサンプルコードはこちらのコードが前提にあります。

```py
import os
from dotenv import load_dotenv

from oci.config import from_file
from oci.generative_ai import GenerativeAiClient
from oci.generative_ai_inference import GenerativeAiInferenceClient
from oci.generative_ai_inference.models import (
    EmbedTextDetails,
    OnDemandServingMode,
    GenerateTextDetails,
    CohereLlmInferenceRequest,
    SummarizeTextDetails
)

load_dotenv()
config = from_file()

GEN_AI_ENDPOINT = os.getenv('GEN_AI_ENDPOINT')
GEN_AI_INFERENCE_ENDPOINT = os.getenv('GEN_AI_INFERENCE_ENDPOINT')
COMPARTMENT_ID = os.getenv('COMPARTMENT_ID')
gen_ai_client = GenerativeAiClient(config=config, service_endpoint=GEN_AI_ENDPOINT)
gen_ai_inference_client = GenerativeAiInferenceClient(config=config, service_endpoint=GEN_AI_INFERENCE_ENDPOINT)
```

特定コンパートメント下で使用可能な全モデルの一覧を出力します。

```py
list_models_response = gen_ai_client.list_models(compartment_id=COMPARTMENT_ID)
list_models_response.data
```

実行結果:

```json
{
  "items": [
    {
      "base_model_id": null,
      "capabilities": ["TEXT_GENERATION"],
      "compartment_id": null,
      "defined_tags": {},
      "display_name": "meta.llama-2-70b-chat",
      "fine_tune_details": null,
      "freeform_tags": {},
      "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyai3pxxkeezogygojnayizqu3bgslgcn6yiqvmyu3w75ma",
      "is_long_term_supported": null,
      "lifecycle_details": "Creating Base Model",
      "lifecycle_state": "ACTIVE",
      "model_metrics": null,
      "system_tags": {},
      "time_created": "2024-01-05T02:19:51.103000+00:00",
      "time_deprecated": null,
      "type": "BASE",
      "vendor": "meta",
      "version": "1.0"
    },
    {
      "base_model_id": null,
      "capabilities": ["TEXT_GENERATION", "FINE_TUNE"],
      "compartment_id": null,
      "defined_tags": {},
      "display_name": "cohere.command",
      "fine_tune_details": null,
      "freeform_tags": {},
      "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya4bk5foxjdxe6332b6dmkzrdtqdulyodbek7ewzt5gqnq",
      "is_long_term_supported": null,
      "lifecycle_details": "BaseModel created",
      "lifecycle_state": "ACTIVE",
      "model_metrics": null,
      "system_tags": {},
      "time_created": "2024-01-02T21:06:03.031000+00:00",
      "time_deprecated": null,
      "type": "BASE",
      "vendor": "cohere",
      "version": "15.6"
    },
    {
      "base_model_id": null,
      "capabilities": ["TEXT_GENERATION", "FINE_TUNE"],
      "compartment_id": null,
      "defined_tags": {},
      "display_name": "cohere.command-light",
      "fine_tune_details": null,
      "freeform_tags": {},
      "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyacw6jlwbfskxv5e4yqd5yupgaxcn6x4q3esrv7punjo5a",
      "is_long_term_supported": null,
      "lifecycle_details": "BaseModel created",
      "lifecycle_state": "ACTIVE",
      "model_metrics": null,
      "system_tags": {},
      "time_created": "2024-01-02T20:50:08.484000+00:00",
      "time_deprecated": null,
      "type": "BASE",
      "vendor": "cohere",
      "version": "15.6"
    },
    {
      "base_model_id": null,
      "capabilities": ["TEXT_SUMMARIZATION"],
      "compartment_id": null,
      "defined_tags": {},
      "display_name": "cohere.command",
      "fine_tune_details": null,
      "freeform_tags": {},
      "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya3yvtzkwd7y4lq6cl3mb22obgesqh2k4k7t2ruzix55ia",
      "is_long_term_supported": null,
      "lifecycle_details": "Base Model created",
      "lifecycle_state": "ACTIVE",
      "model_metrics": null,
      "system_tags": {},
      "time_created": "2023-12-01T20:48:50.329000+00:00",
      "time_deprecated": null,
      "type": "BASE",
      "vendor": "cohere",
      "version": "15.6"
    },
    {
      "base_model_id": null,
      "capabilities": ["TEXT_GENERATION"],
      "compartment_id": null,
      "defined_tags": {},
      "display_name": "cohere.command",
      "fine_tune_details": null,
      "freeform_tags": {},
      "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyafhwal37hxwylnpbcncidimbwteff4xha77n5xz4m7p6a",
      "is_long_term_supported": null,
      "lifecycle_details": "Base Model created",
      "lifecycle_state": "ACTIVE",
      "model_metrics": null,
      "system_tags": {},
      "time_created": "2023-12-01T20:46:38.865000+00:00",
      "time_deprecated": null,
      "type": "BASE",
      "vendor": "cohere",
      "version": "15.6"
    },
    {
      "base_model_id": null,
      "capabilities": ["TEXT_GENERATION"],
      "compartment_id": null,
      "defined_tags": {},
      "display_name": "cohere.command-light",
      "fine_tune_details": null,
      "freeform_tags": {},
      "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyad4uybcjtw475o4uqja3oz3kzrfhln7joysjxj3npp2la",
      "is_long_term_supported": null,
      "lifecycle_details": "Base Model created",
      "lifecycle_state": "ACTIVE",
      "model_metrics": null,
      "system_tags": {},
      "time_created": "2023-12-01T20:42:45.284000+00:00",
      "time_deprecated": null,
      "type": "BASE",
      "vendor": "cohere",
      "version": "15.6"
    },
    {
      "base_model_id": null,
      "capabilities": ["TEXT_EMBEDDINGS"],
      "compartment_id": null,
      "defined_tags": {},
      "display_name": "cohere.embed-english-light-v3.0",
      "fine_tune_details": null,
      "freeform_tags": {},
      "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya6fw6mqgi2mcmehwvd7h2ysqe25rbhprgclypiyh46wra",
      "is_long_term_supported": null,
      "lifecycle_details": "Base Model created",
      "lifecycle_state": "ACTIVE",
      "model_metrics": null,
      "system_tags": {},
      "time_created": "2023-11-30T20:55:12.928000+00:00",
      "time_deprecated": null,
      "type": "BASE",
      "vendor": "cohere",
      "version": "3.0"
    },
    {
      "base_model_id": null,
      "capabilities": ["TEXT_EMBEDDINGS"],
      "compartment_id": null,
      "defined_tags": {},
      "display_name": "cohere.embed-english-v3.0",
      "fine_tune_details": null,
      "freeform_tags": {},
      "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya3bqursz5i2eeg5eesvnlrqj4mrdmi3infd4ve3kaqjva",
      "is_long_term_supported": null,
      "lifecycle_details": "Base Model created",
      "lifecycle_state": "ACTIVE",
      "model_metrics": null,
      "system_tags": {},
      "time_created": "2023-11-30T20:54:14.931000+00:00",
      "time_deprecated": null,
      "type": "BASE",
      "vendor": "cohere",
      "version": "3.0"
    },
    {
      "base_model_id": null,
      "capabilities": ["TEXT_EMBEDDINGS"],
      "compartment_id": null,
      "defined_tags": {},
      "display_name": "cohere.embed-multilingual-light-v3.0",
      "fine_tune_details": null,
      "freeform_tags": {},
      "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyaya37tnqbhe7fdikai6r2z5qfb3aq6mdg3z3kbeenmlha",
      "is_long_term_supported": null,
      "lifecycle_details": "Base Model created",
      "lifecycle_state": "ACTIVE",
      "model_metrics": null,
      "system_tags": {},
      "time_created": "2023-11-30T20:52:21.320000+00:00",
      "time_deprecated": null,
      "type": "BASE",
      "vendor": "cohere",
      "version": "3.0"
    },
    {
      "base_model_id": null,
      "capabilities": ["TEXT_EMBEDDINGS"],
      "compartment_id": null,
      "defined_tags": {},
      "display_name": "cohere.embed-multilingual-v3.0",
      "fine_tune_details": null,
      "freeform_tags": {},
      "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyadcwe6y5jt655ox5hkm76vq2wz3hs7ljbfhljvithlnoq",
      "is_long_term_supported": null,
      "lifecycle_details": "Base Model created",
      "lifecycle_state": "ACTIVE",
      "model_metrics": null,
      "system_tags": {},
      "time_created": "2023-11-30T20:51:06.303000+00:00",
      "time_deprecated": null,
      "type": "BASE",
      "vendor": "cohere",
      "version": "3.0"
    },
    {
      "base_model_id": null,
      "capabilities": ["TEXT_GENERATION", "FINE_TUNE"],
      "compartment_id": null,
      "defined_tags": {},
      "display_name": "cohere.command",
      "fine_tune_details": null,
      "freeform_tags": {},
      "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaapi24rzaaxh5ulr23qhwme4bbix3bu3x5opqswkaxpuirgcmo4m7a",
      "is_long_term_supported": null,
      "lifecycle_details": "Created BaseModel",
      "lifecycle_state": "ACTIVE",
      "model_metrics": null,
      "system_tags": {},
      "time_created": "2023-10-13T21:21:33.212000+00:00",
      "time_deprecated": null,
      "type": "BASE",
      "vendor": "cohere",
      "version": "14.2"
    },
    {
      "base_model_id": null,
      "capabilities": ["TEXT_GENERATION", "FINE_TUNE"],
      "compartment_id": null,
      "defined_tags": {},
      "display_name": "cohere.command-light",
      "fine_tune_details": null,
      "freeform_tags": {},
      "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaapi24rzaalcbnpexqvkxpwfdcqskcupap27qtvr7ew5ierjpbih2a",
      "is_long_term_supported": null,
      "lifecycle_details": "Created",
      "lifecycle_state": "ACTIVE",
      "model_metrics": null,
      "system_tags": {},
      "time_created": "2023-09-08T22:20:13.843000+00:00",
      "time_deprecated": null,
      "type": "BASE",
      "vendor": "cohere",
      "version": "14.2"
    },
    {
      "base_model_id": null,
      "capabilities": ["TEXT_EMBEDDINGS"],
      "compartment_id": null,
      "defined_tags": {},
      "display_name": "cohere.embed-english-light-v2.0",
      "fine_tune_details": null,
      "freeform_tags": {},
      "id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaapi24rzaauhcci2xuk7263dbon4xnkjku5ifv6i45nzbz6rrn4joa",
      "is_long_term_supported": null,
      "lifecycle_details": "Created",
      "lifecycle_state": "ACTIVE",
      "model_metrics": null,
      "system_tags": {},
      "time_created": "2023-09-08T18:58:28.048000+00:00",
      "time_deprecated": null,
      "type": "BASE",
      "vendor": "cohere",
      "version": "2.0"
    }
  ]
}
```

### GenerativeAiInferenceClient

メソッドとしては、以下が提供されています。SDK の方はさすがにテキスト生成も提供されているようです。

| **init**(config, \*\*kwargs)                       | Creates a new service client                        |
| -------------------------------------------------- | --------------------------------------------------- |
| embed_text(embed_text_details, \*\*kwargs)         | Produces embeddings for the inputs.                 |
| generate_text(generate_text_details, \*\*kwargs)   | Generates a text response based on the user prompt. |
| summarize_text(summarize_text_details, \*\*kwargs) | Summarizes the input text.                          |

※[oci.generative_ai_inference.GenerativeAiInferenceClient](https://docs.oracle.com/en-us/iaas/tools/python/2.119.1/api/generative_ai_inference/client/oci.generative_ai_inference.GenerativeAiInferenceClient.html#oci.generative_ai_inference.GenerativeAiInferenceClient) から引用

順番に見ていきます。

#### embed_text

ユーザーの入力に対する埋め込み表現を得るためのメソッドです。RAG 等でベクトル検索をする際にユーザーの入力を埋め込み表現とするために使うこと等が想定されます。

```py
inputs = [
    "Learn about the Employee Stock Purchase Plan",
    "Reassign timecard approvals during leave",
    "View my payslip online",
    "Learn about the Employee Stock Purchase Plan",
    "Reassign timecard approvals during leave",
    "View my payslip online",
    "Enroll in benefits",
    "Change my direct deposit",
    "Have my employment/income verified",
    "Request A Workplace Accommodation",
    "Submit my time card",
    "Report Information Security Incidents",
    "Review the Code of Conduct",
    "Review the Social Media Policy",
    "Review Corporate Information Security Policies",
    "Understand Compliance and Ethics",
    "Understand the Fiscal Year Calendar",
    "Change my email address",
    "Change my personal information",
    "Learn about Analyst Relations",
    "Learn about Business Skills",
    "Learn about Career Development",
    "Learn about Employee Resource Groups",
    "Learn about Information Security",
    "Learn about Leadership skills",
    "Learn about sustainability",
    "Learn about Technical skills",
    "Request a Phone Number",
    "Learn about video conferencing",
    "Learn about Organizational Distribution Lists",
    "Subscribe to Group Mailing Lists",
    "Find a Campus Map",
    "Obtain a security badge",
    "Obtain an office workspace",
    "Submit an ergonomics request",
    "Use printers",
    "Delegate workflows or transaction approvals while out on leave",
    "Tips for working remotely",
    "Volunteer",
    "Reassign workflow approvals while on vacation",
    "How to delegate timecard approvals",
    "Apply for Corporate Credit Card",
    "Book travel",
    "Submit an expense report"
]

embed_text_result = gen_ai_inference_client.embed_text(
    embed_text_details=EmbedTextDetails(
        inputs=inputs,
        serving_mode=OnDemandServingMode(model_id=EMBEDDINGS_MODEL_OCID),
        compartment_id=COMPARTMENT_ID,
        is_echo=True,
        truncate='NONE',
    )
)

embed_text_result.data
```

実行結果:

とてつもなく長いので、割愛します。ご自身でお試しください！

#### generate_text

ユーザーの入力に対してテキスト生成を行うためのメソッドです。以下のコードは、JD(Job Description)を生成するサンプルコードです。

```py
prompt = """
Generate a job description for a data visualization expert with the following three qualifications only:
1) At least 5 years of data visualization expert
2) A great eye for details
3) Ability to create original visualizations
"""

generate_text_response = gen_ai_inference_client.generate_text(
    generate_text_details=GenerateTextDetails(
        compartment_id=COMPARTMENT_ID,
        serving_mode=OnDemandServingMode(
            model_id=GENERATION_MODEL_OCID
        ),
        inference_request=CohereLlmInferenceRequest(
            prompt=prompt,
            is_stream=False,
            num_generations=1
        )
    )
)

generate_text_response.data
```

実行結果:

```json
{
  "inference_response": {
    "generated_texts": [
      {
        "finish_reason": null,
        "id": "1566d2c8-dfc0-498c-a5c7-bc71023cf79f",
        "likelihood": null,
        "text": " We're looking for a talented Data Visualization Expert to join our team! The ideal candidate will have",
        "token_likelihoods": null
      }
    ],
    "prompt": null,
    "runtime_type": "COHERE",
    "time_created": "2024-01-29T10:39:42.604000+00:00"
  },
  "model_id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceyafhwal37hxwylnpbcncidimbwteff4xha77n5xz4m7p6a",
  "model_version": "15.6"
}
```

#### summarize_text

与えられた入力に対して、その要約を作成するためのメソッドです。以下のコードはブログの内容を要約するサンプルコードです。

```py
input = """
Oracle’s strategy is built around the reality that enterprises work with AI through three different modalities: Infrastructure, models and services, and within applications.

First, we provide a robust infrastructure for training and serving models at scale. Through our partnership with NVIDIA, we can give customers superclusters, which are powered by the latest GPUs in the market connected together with an ultra-low-latency RDMA over converged ethernet (RoCE) network. This solution provides a highly performant, cost-effective method for training generative AI models at scale. Many AI startups like Adept and MosaicML are building their products directly on OCI.

Second, we provide easy-to-use cloud services for developers and scientists to utilize in fully managed implementations. We’re enabling new generative AI services and business functions through our partnership with Cohere, a leading generative AI company for enterprise-grade large language models (LLMs). Through our partnership with Cohere, we’re building a new generative AI service. This upcoming AI service, OCI Generative AI, enables OCI customers to add generative AI capabilities to their own applications and workflows through simple APIs.

Third, we embed generative models into the applications and workflows that business users use every day. Oracle plans to embed generative AI from Cohere into its Fusion, NetSuite, and our vertical software-as-a-service (SaaS) portfolio to create solutions that provide organizations with the full power of generative AI immediately. Across industries, Oracle can provide native generative AI-based features to help organizations automate key business functions, improve decision-making, and enhance customer experiences. For example, in healthcare, Oracle Cerner manages billions of electronic health records (EHR). Using anonymized data, Oracle can create generative models adapted to the healthcare domain, such as automatically generating a patient discharge summary or a letter of authorization for medical insurance.

Oracle’s generative AI offerings span applications to infrastructure and provide the highest levels of security, performance, efficiency, and value.
"""

summarize_text_response = gen_ai_inference_client.summarize_text(
    summarize_text_details=SummarizeTextDetails(
        input=input,
        serving_mode=OnDemandServingMode(
            model_id=SUMMARIZE_MODEL_OCID
        ),
        compartment_id=COMPARTMENT_ID,
        is_echo=True,
        temperature=1.0,
        length='SHORT',
        format='AUTO',
        extractiveness='AUTO'
    )
)

summarize_text_response.data
```

実行結果:

```json
{
  "id": "9ca1129e-8d48-41df-bc46-1caaf973e718",
  "input": "\nOracle\u2019s strategy is built around the reality that enterprises work with AI through three different modalities: Infrastructure, models and services, and within applications.\n\nFirst, we provide a robust infrastructure for training and serving models at scale. Through our partnership with NVIDIA, we can give customers superclusters, which are powered by the latest GPUs in the market connected together with an ultra-low-latency RDMA over converged ethernet (RoCE) network. This solution provides a highly performant, cost-effective method for training generative AI models at scale. Many AI startups like Adept and MosaicML are building their products directly on OCI.\n\nSecond, we provide easy-to-use cloud services for developers and scientists to utilize in fully managed implementations. We\u2019re enabling new generative AI services and business functions through our partnership with Cohere, a leading generative AI company for enterprise-grade large language models (LLMs). Through our partnership with Cohere, we\u2019re building a new generative AI service. This upcoming AI service, OCI Generative AI, enables OCI customers to add generative AI capabilities to their own applications and workflows through simple APIs.\n\nThird, we embed generative models into the applications and workflows that business users use every day. Oracle plans to embed generative AI from Cohere into its Fusion, NetSuite, and our vertical software-as-a-service (SaaS) portfolio to create solutions that provide organizations with the full power of generative AI immediately. Across industries, Oracle can provide native generative AI-based features to help organizations automate key business functions, improve decision-making, and enhance customer experiences. For example, in healthcare, Oracle Cerner manages billions of electronic health records (EHR). Using anonymized data, Oracle can create generative models adapted to the healthcare domain, such as automatically generating a patient discharge summary or a letter of authorization for medical insurance.\n\nOracle\u2019s generative AI offerings span applications to infrastructure and provide the highest levels of security, performance, efficiency, and value.\n",
  "model_id": "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya3yvtzkwd7y4lq6cl3mb22obgesqh2k4k7t2ruzix55ia",
  "model_version": "15.6",
  "summary": "Oracle has developed an AI strategy that recognizes the different ways in which enterprises interact with AI, focusing on infrastructure, model and services, and applications. To achieve this, they partner with NVIDIA to provide affordable GPUs to train large AI models at scale, and with Cohere, a leading LLM company, to develop and embed generative AI into their SaaS portfolio to automate business functions, improve decisions and enhance customer experience. Oracle seeks to provide high levels of security, performance, efficiency and value across its generative AI offerings. \n\nWould you like help with anything else regarding Oracle?"
}
```

# 参考情報

https://docs.oracle.com/ja-jp/iaas/Content/generative-ai/home.htm

https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.37.5/oci_cli_docs/cmdref/generative-ai.html

https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.37.5/oci_cli_docs/cmdref/generative-ai-inference.html

https://docs.oracle.com/en-us/iaas/tools/python/2.119.1/api/generative_ai.html

https://docs.oracle.com/en-us/iaas/tools/python/2.119.1/api/generative_ai_inference.html
