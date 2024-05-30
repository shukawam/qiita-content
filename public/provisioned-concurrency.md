---
title: Oracle Functions の新機能 - Provisioned Concurrency について
tags:
  - oci
  - oraclecloud
  - Functions
private: false
updated_at: '2022-05-10T11:11:31+09:00'
id: ec128cc3dd1b169cc3e6
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

Oracle Functions に限らず、FaaS と呼ばれる実行環境では、実行時に動的にリソースを割り当てる事で柔軟なリソース利用（主に、コストやスケール）を実現しています。一方で、初回の実行時には計算リソースや NW リソースの割り当てにある程度時間がかかるので、レスポンスが遅くなる Cold Start 問題は、FaaS を採用する上で避けては通れない問題です。従来では「Functions が再度 Cold Start を引かないような間隔で定期的にリクエストを送り続ける」という手法でこの問題を回避していましたが、今回の機能アップデートでもう少し素直に Cold Start 問題に対処できるようになりました。

# Provisioned Concurrency とは？

関数の実行前に決められた数の Function の各種リソース割り当てを済ませておくことで初回実行時の性能を向上させるための機能です。すごくざっくりとしたイメージが以下のようになります。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/44807b5d-a862-1ddb-964b-d10fd111c95a.png)

図中に出てきた PCUs という単位は、Provisioned Concurrency Units と呼ばれるもので、事前にどれくらいの数のリソース割り当てを完了させておきますか？という設定値になります。この数は、任意の数を設定できるというわけではなく Functions に割り当てられたメモリ量に応じて設定ができるようになっています。設定可能な PCUs は以下の通りです。（リージョン毎に 40GB まで割り当てることが可能）

| Functions に割り当てられたメモリ量 | PCUs   |
| ---------------------------------- | ------ |
| 128 MB                             | 40 x n |
| 256 MB                             | 20 x n |
| 512 MB                             | 10 x n |
| 1024 MB                            | 10 x n |
| 2048 MB                            | 10 x n |

# 設定方法

## 前提

- OCI の有償アカウントを有していること
- Fn CLI/OCI CLI の最新版がインストール済みであること
  - 手元にない場合は、Cloud Shell を使うと良いです
- Functions を使用するための初期設定が完了していること
  - 完了していない場合は、こちらを参照ください
    - [Fn Project ハンズオン](https://oracle-japan.github.io/ocitutorials/cloud-native/fn-for-beginners/)
    - [Oracle Functions ハンズオン](https://oracle-japan.github.io/ocitutorials/cloud-native/functions-for-beginners/)

## 設定手順

設定手順は非常に簡単です。まずは、適当な Function を Fn CLI で作成します。

```bash
fn init --runtime go provisioned-concurrency-test-func
```

実行結果

```bash
Creating function at: ./provisioned-concurrency-test-func
Function boilerplate generated.
func.yaml created.
```

今回は中身を特に変えずにこのままデプロイしたいと思います。まずは Function のビルドをします。

```bash
cd provisioned-concurrency-test-func; \
fn build
```

`Function nrt.ocir.io/<namespace>/<repo>/provisioned-concurrency-test-func:0.0.1 built successfully.` のようなメッセージが出力されればビルドは完了です。次に、OCIR へプッシュします。

```bash
fn push
```

`Function nrt.ocir.io/<namespace>/<repo>/provisioned-concurrency-test-func:0.0.1 pushed successfully to Docker Hub.` のように出力されれば OCIR へのプッシュは完了です。次に、Oracle Functions の論理単位であるアプリケーションを作成します。（\<compartment-ocid\>, \<subnet-ocid\>は、ご自身の環境に合わせて設定してください）

```bash
oci fn application create \
--compartment-id <compartment-ocid> \
--display-name provisioned-concurrency \
--subnet-ids '["<subnet-ocid>"]'
```

実行結果（一部マスクしています）

```bash
{
  "data": {
    "compartment-id": "<compartment-ocid>",
    "config": {},
    "defined-tags": {},
    "display-name": "provisioned-concurrency",
    "freeform-tags": {},
    "id": "<application-ocid>",
    "image-policy-config": null,
    "lifecycle-state": "ACTIVE",
    "network-security-group-ids": [],
    "subnet-ids": [
      "<subnet-ocid>"
    ],
    "syslog-url": "",
    "time-created": "2022-05-09T15:01:41.382000+00:00",
    "time-updated": "2022-05-09T15:01:41.382000+00:00",
    "trace-config": {
      "domain-id": "",
      "is-enabled": false
    }
  },
  "etag": "ad873df8dc37904858c7f6c2134276bd95f6e51775f43a786008bfb4b6cc6f64"
}
```

次に、作成したアプリケーションで管理する Function を作成します。

```bash
oci fn function create \
--application-id <application-ocid> \
--display-name provisioned-concurrency-test-func \
--image nrt.ocir.io/<namespace>/<repo>/provisioned-concurrency-test-func:0.0.1 \
--provisioned-concurrency "{\"strategy\": \"CONSTANT\", \"count\": 40}" \
--memory-in-mbs 128
```

実行結果（一部、マスクしています）

```bash
{
  "data": {
    "application-id": "<application-ocid>",
    "compartment-id": "<compartment-ocid>",
    "config": {},
    "defined-tags": {},
    "display-name": "provisioned-concurrency-test-func",
    "freeform-tags": {},
    "id": "<function-ocid>",
    "image": "nrt.ocir.io/<namespace>/<repo>/provisioned-concurrency-test-func:0.0.1",
    "image-digest": "sha256:5b53a7ac483dbe268c5f6b5e5de322b62cbf3449a084302d217c5a2a282ff176",
    "invoke-endpoint": "https://...ap-tokyo-1.functions.oci.oraclecloud.com",
    "lifecycle-state": "ACTIVE",
    "memory-in-mbs": 128,
    "provisioned-concurrency-config": {
      "count": 40,
      "strategy": "CONSTANT"
    },
    "time-created": "2022-05-09T15:08:49.352000+00:00",
    "time-updated": "2022-05-09T15:08:49.352000+00:00",
    "timeout-in-seconds": 30,
    "trace-config": {
      "is-enabled": false
    }
  },
  "etag": "ba0b922a36e9c5817ef790bd80f7e6f94b3546836267dcaa2bd45c25c764d078"
}
```

これで、アプリケーション、Function が作成されました。一応、確認しておきます。

アプリケーション

```bash
fn list apps | grep -i provisioned
```

実行結果

```bash
provisioned-concurrency <application-ocid>
```

Function

```bash
fn list functions provisioned-concurrency | grep -i provisioned
```

実行結果

```bash
provisioned-concurrency-test-func       nrt.ocir.io/<namespace>/<repo>/provisioned-concurrency-test-func:0.0.1       <function-ocid>
```

次に、作成した Function を実行してみます。

```bash
time fn invoke provisioned-concurrency provisioned-concurrency-test-func
```

実行結果

```bash
{"message":"Hello World"}

real    0m0.826s
user    0m0.126s
sys     0m0.020s
```

以上です。

## ちなみに

Provisioned Concurrency の設定を抜いて実行してみます。

```bash
oci fn function update \
--function-id <function-ocid> \
--provisioned-concurrency "{\"strategy\": \"NONE\"}"
```

実行結果

```bash
{
  "data": {
    "application-id": "<application-ocid>",
    "compartment-id": "<compartment-ocid>",
    "config": {},
    "defined-tags": {},
    "display-name": "provisioned-concurrency-test-func",
    "freeform-tags": {},
    "id": "<function-ocid>",
    "image": "nrt.ocir.io/<namespace>/<repo>/provisioned-concurrency-test-func:0.0.1",
    "image-digest": "sha256:5b53a7ac483dbe268c5f6b5e5de322b62cbf3449a084302d217c5a2a282ff176",
    "invoke-endpoint": "https://...ap-tokyo-1.functions.oci.oraclecloud.com",
    "lifecycle-state": "ACTIVE",
    "memory-in-mbs": 128,
    "provisioned-concurrency-config": {
      "strategy": "NONE"
    },
    "time-created": "2022-05-09T15:08:49.352000+00:00",
    "time-updated": "2022-05-09T15:20:06.769000+00:00",
    "timeout-in-seconds": 30,
    "trace-config": {
      "is-enabled": false
    }
  },
  "etag": "44fa9d4d6584a48ea6a3bfb0b40f16eb304c9f8c389b8966c5d76674ef61d31b"
}
```

（ある程度時間を置き）再度、実行してみます。

```bash
time fn invoke provisioned-concurrency provisioned-concurrency-test-func
```

実行結果

```bash
{"message":"Hello World"}

real    0m1.548s
user    0m0.103s
sys     0m0.045s
```

確かに、Provisioned Concurrency の設定を入れた方が初回実行が早くなっていることが確認できました。

# おわりに

ちょっとした設定で簡単に Cold Start 問題の対策ができるので是非試してみてください！

# 参考情報

- [Reducing Initial Latency Using Provisioned Concurrency](https://docs.oracle.com/en-us/iaas/Content/Functions/Tasks/functionsusingprovisionedconcurrency.htm)
