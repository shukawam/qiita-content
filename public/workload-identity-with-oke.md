---
title: Workload Identity w/ OKE
tags:
  - kubernetes
  - oci
private: false
updated_at: '2024-05-17T17:18:49+09:00'
id: fb419894179cc07c8ba6
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

OKE[^1] の Workload Identity を試してみます。

[^1]: Oracle Container Engine for Kubernetes

# Workload Identity とはなにものか？

従来、OKE にデプロイしたアプリケーションから OCI のリソースを操作するときの認証・認可の制御は API キーを使うかインスタンス・プリンシパル[^2]を使います。API キーは、IAM ユーザーに紐づくため、異なるアクセス要件を持つ Pod に異なるポリシーが付与されたユーザーから発行された API キーを含めれば、Pod 毎のアクセス制御は技術的にはできると思います。しかし、異なる要件ごとにクラウド側の IAM ユーザーが増えていくのに加え、Pod に対していちいち API キーをマウントしてあげないといけないため、そもそも面倒ですし仮に API キーが流出してしまった場合は、新しく API キーを発行してそれぞれの Pod に再度マウントして...と取り回しがかなり面倒です。
そのため、多くの場合はインスタンス・プリンシパルを使ってアクセス制御を行いますが、こちらは Kubernetes のノード単位（正確には、Kubernetes を構築するノードが含まれている動的グループ）にポリシー制御を行うため、同じノード内で異なるリソースアクセスの要件を持つ Pod に対して細かな制御をすることができません。

[^2]: OCI の Compute Instance を含む動的グループを作成し、そのグループに対してポリシーを設定することでアクセス制御を実現する方法のこと

しかし、2023 年の 3 月に Workload Identity という機能が OKE でサポートされました。

https://docs.oracle.com/en-us/iaas/releasenotes/changes/09839612-f2a4-41df-9a57-f81549678e17/

一言で表すなら、”OKE 上で稼働するワークロード自体が IAM ポリシーのリソースとして扱うことができるようになった”と言うことができます。つまり、API キーのような Kubernetes 上で扱うなら取り回しし辛いものを使うことなく、ワークロード単位（≒Pod 単位）のアクセス制御が実現できるようになった！ということです。

また、当該機能は、拡張クラスタのみでサポートされており、SDK に関しては Go, Java, Python の特定バージョン以上のものをサポートしているとドキュメントに記載がありますので、使用する場合はこのような前提があることを留意してください。

https://docs.oracle.com/ja-jp/iaas/Content/ContEng/Tasks/contenggrantingworkloadaccesstoresources.htm

:::note info
ドキュメントには、対応 SDK は Go, Java, Python と記載がありますが、GitHub を見てみると少なくとも TypeScript と Ruby については対応しているように見受けられます。

https://github.com/oracle/oci-typescript-sdk/blob/master/lib/common/lib/auth/oke-workload-identity-authentication-details-provider.ts

https://github.com/oracle/oci-ruby-sdk/blob/master/lib/oci/auth/signers/oke_workload_identity_resource_principal_signer.rb
:::

# どのように動作する？

[Workload Identity はなにものか？](#workload-identity-はなにものか)で簡単に説明しましたが、今回例として作成する環境を元にもう少し深掘りしてどのように動作するのか見ていきましょう。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/61beddc3-6973-9a07-322c-80824687d96a.png)

上図のように、クラスタには _admin_ と _viewer_ というワークロードが存在するとします。このとき、_admin_ は Object Storage に対して `manage` 以下の全ての操作（`manage`, `use`, `read`, `inspect`）を許可し、_viewer_ は、Object Storage に対して `read` 以下の全ての操作（`read`, `inspect`）を許可するとします。
この時、実際のアクセス制御は IAM ポリシーの where 句に対象のワークロードを指定することで実現します。Kubernetes には、クラスタ内部で個別の ID を提供する Service Account という仕組みが存在するため、IAM ポリシーにはこの Service Account を OCI 側から特定するための情報（クラスタの OCID、Namespace、Service Account の名前）を与えれば良いわけです。実際のポリシーの構文を見てみると、以下のようになります。

```sh
Allow any-user to <verb> <resource> in <location>
where all {
  request.principal.type = 'workload',
  request.principal.cluster_id = '<cluster-ocid>',
  request.principal.namespace = '<namespace-name>',
  request.principal.service_account = '<service-account-name>'
}
```

Go の場合は、以下のように Workload Identity を使うための Authentication Provider を宣言し、それを使用するクライアントに渡します。（Java の場合は少しお作法が違うので、別の記事でも解説したいと思います）

```go
provider, err := auth.OkeWorkloadIdentityConfigurationProvider()
if err != nil {
  // do something
}
client, clerr := core.NewComputeClientWithConfigurationProvider(provider)
if clerr != nil {
  // do something
}
```

# 実際に試してみる

今回は、OKE の拡張クラスタが作成済みであることを前提に書きます。また、本記事の動作環境は以下の通りです。

- Kubernetes v1.29.1
- Go 1.22.2
- SDK for Go v65.65.1

## Namespace, Service Account の作成

ワークロードをデプロイするための Namespace とそのワークロードを識別するための Service Account を作成します。

Namespace の作成

```sh
kubectl create namespace example
```

Service Account の作成（_admin_）

```sh
kubectl create sa admin -n example
```

Service Account の作成（_viewer_）

```sh
kubectl create sa viewer -n example
```

## IAM ポリシーの作成

OCI CLI を用いて作成しますが、コンソールから実施しても同様です。

以下の JSON を自身の環境に合わせて修正します。

```create-workload-identity-policy.json
{
  "compartmentId": "<compartment-id>",
  "description": "Policy for OKE Workload Identity.",
  "name": "workload-identity-policy",
  "statements": [
    "Allow any-user to manage buckets in tenancy where all {request.principal.type = 'workload', request.principal.cluser_id = '<oke-cluster-id>', request.principal.namespace = 'example', request.principal.service_account = 'admin'}",
    "Allow any-user to read buckets in tenancy where all {request.principal.type = 'workload', request.principal.cluser_id = '<oke-cluster-id>', request.principal.namespace = 'example', request.principal.service_account = 'viewer'}"
  ]
}
```

作成した JSON ファイルを用いて、IAM Policy を作成します。

```sh
oci iam policy create --from-json file://create-workload-identity-policy.json
```

## アプリケーションの実装

今回は、[Gin](https://github.com/gin-gonic/gin) で実装した Web アプリケーションをテストに用います。実装は以下の通りです。

```go
package main

import (
  "context"
  "fmt"
  "net/http"
  "os"

  "github.com/gin-gonic/gin"
  "github.com/oracle/oci-go-sdk/v65/common/auth"
  "github.com/oracle/oci-go-sdk/v65/core"
  "github.com/oracle/oci-go-sdk/v65/objectstorage"
)

func main() {
  r := gin.Default()
  r.GET("/os/bucket", listBucket)
  r.POST("/os/bucket/:bucketName", createBucket)
  r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")
}

func listBucket(c *gin.Context) {
  provider, err := auth.OkeWorkloadIdentityConfigurationProvider()
  if err != nil {
    panic(err)
  }
  fmt.Println(provider)
  client, clerr := objectstorage.NewObjectStorageClientWithConfigurationProvider(provider)
  if clerr != nil {
    panic(clerr)
  }
  region := os.Getenv("OCI_RESOURCE_PRINCIPAL_REGION")
  namespace := os.Getenv("NAMESPACE")
  compartmentId := os.Getenv("COMPARTMENT_ID")
  client.SetRegion(region)
  res, rerr := client.ListBuckets(
    c,
    objectstorage.ListBucketsRequest{
      NamespaceName: &namespace,
      CompartmentId: &compartmentId,
    },
  )
  if rerr != nil {
    panic(rerr)
  }
  c.JSON(
    http.StatusOK,
    gin.H{
      "data": res.Items,
    },
  )
}

func createBucket(c *gin.Context) {
  provider, err := auth.OkeWorkloadIdentityConfigurationProvider()
  if err != nil {
    panic(err)
  }
  fmt.Println(provider)
  client, clerr := objectstorage.NewObjectStorageClientWithConfigurationProvider(provider)
  if clerr != nil {
    panic(clerr)
  }
  region := os.Getenv("OCI_RESOURCE_PRINCIPAL_REGION")
  namespace := os.Getenv("NAMESPACE")
  compartmentId := os.Getenv("COMPARTMENT_ID")
  bucketName := c.Param("bucketName")
  client.SetRegion(region)
  res, rerr := client.CreateBucket(
    c,
    objectstorage.CreateBucketRequest{
      NamespaceName: &namespace,
      CreateBucketDetails: objectstorage.CreateBucketDetails{
        Name:          &bucketName,
        CompartmentId: &compartmentId,
      },
    },
  )
  if rerr != nil {
    panic(rerr)
  }
  c.JSON(
    http.StatusOK,
    gin.H{
      "data": res.Bucket,
    },
  )
}
```

いくつか、環境変数から設定を読み込んでいます。今回は、OKE 上で実行するため Manifest に環境変数を定義しています。

```admin.yaml
apiVersion: v1
kind: Service
metadata:
  name: admin
  labels:
    app: admin
spec:
  type: ClusterIP
  selector:
    app: admin
  ports:
    - port: 8080
      targetPort: 8080
      name: http
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: admin
  namespace: example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admin
  template:
    metadata:
      labels:
        app: admin
        version: v1
    spec:
      serviceAccountName: admin
      automountServiceAccountToken: true
      containers:
        - name: admin
          image: lhr.ocir.io/orasejapan/shukawam/workload-identity-test:0.0.1
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          env:
            - name: OCI_RESOURCE_PRINCIPAL_VERSION
              value: "2.2"
            - name: OCI_RESOURCE_PRINCIPAL_REGION
              value: "ap-tokyo-1"
            - name: NAMESPACE
              value: <your-namespace>
            - name: COMPARTMENT_ID
              value: <your-compartment-id>
            - name: OCI_GO_SDK_DEBUG
              value: info
      imagePullSecrets:
        - name: ocir-secret
```

```viewer.yaml
apiVersion: v1
kind: Service
metadata:
  name: viewer
  labels:
    app: viewer
spec:
  type: ClusterIP
  selector:
    app: viewer
  ports:
    - port: 8080
      targetPort: 8080
      name: http
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: viewer
  namespace: example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: viewer
  template:
    metadata:
      labels:
        app: viewer
        version: v1
    spec:
      serviceAccountName: viewer
      automountServiceAccountToken: true
      containers:
        - name: viewer
          image: lhr.ocir.io/orasejapan/shukawam/workload-identity-test:0.0.1
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          env:
            - name: OCI_RESOURCE_PRINCIPAL_VERSION
              value: "2.2"
            - name: OCI_RESOURCE_PRINCIPAL_REGION
              value: "ap-tokyo-1"
            - name: NAMESPACE
              value: <your-namespace>
            - name: COMPARTMENT_ID
              value: <your-compartment-id>
            - name: OCI_GO_SDK_DEBUG
              value: info
      imagePullSecrets:
        - name: ocir-secret
```

:::note warn
ドキュメントには記載がないのですが、OCI SDK for Go の場合、`OCI_RESOURCE_PRINCIPAL_VERSION`, `OCI_RESOURCE_PRINCIPAL_REGION` を指定しないと当該環境変数が必要だというエラーログが出力されます。GitHub Issue にその旨が記載されていたので、ご参考までに。

https://github.com/oracle/oci-go-sdk/issues/489

:::

また、プライベートなレジストリに格納している場合などは、`imagePullSecrets` に対応した Kubernetes Secret を作成ください。

## 動作確認

[アプリケーションの実装](#アプリケーションの実装)でみた通り、アプリケーションには Object Storage - Bucket を参照、作成するためのエンドポイントが 1 つずつ用意されています。この同じアプリケーションに対して、別の Service Account を紐づけることでどのように振る舞いが変化するのかみてみましょう。

まずは、_admin_ から確認してみます。

```sh
kubectl -n example port-forward svc/admin 8080:8080
```

バケット作成のリクエストを送ってみます。

```sh
curl --request POST \
  --url http://localhost:8080/os/bucket/workload-identity \
  --header 'content-type/application/json' | jq
```

実行結果

```json
{
  "data": {
    "namespace": "<namespace>",
    "name": "workload-identity",
    "compartmentId": "<compartment-id>",
    "metadata": {},
    "createdBy": "ocid1.workload.oc1.nrt.ccgtvp5s4ca.mdawmdyynja4ztq3nwfmyjrkmtc5mdywmmqzzwjjzdrhy2i4",
    "timeCreated": "2024-05-17T06:28:24.307Z",
    "etag": "d3c3221c-b68b-4b37-9261-fb76b77c92a7",
    "publicAccessType": "NoPublicAccess",
    "storageTier": "Standard",
    "objectEventsEnabled": false,
    "freeformTags": {},
    "definedTags": {
      "Oracle-Tags": {
        "CreatedBy": "ocid1.workload.oc1.nrt.ccgtvp5s4ca.mdawmdyynja4ztq3nwfmyjrkmtc5mdywmmqzzwjjzdrhy2i4",
        "CreatedOn": "2024-05-17T06:28:24.298Z"
      }
    },
    "kmsKeyId": null,
    "objectLifecyclePolicyEtag": null,
    "approximateCount": null,
    "approximateSize": null,
    "replicationEnabled": false,
    "isReadOnly": false,
    "id": "ocid1.bucket.oc1.ap-tokyo-1.aaaaaaaadtmgjb4rsy6hczowbajk2f27mq6xuidylv5wok4bxkytdjnmgmzq",
    "versioning": "Disabled"
  }
}
```

しっかりとバケットを作成することができました。同様にバケットの一覧も取得することができます。

```sh
curl --request GET \
  --url http://localhost:8080/os/bucket \
  --header 'content-type/application/json' | jq
```

実行結果は以下の通りです。

```json
{
  "data": [
    {
      "namespace": "<namespace>",
      "name": "workload-identity",
      "compartmentId": "<compartment-id>",
      "createdBy": "ocid1.workload.oc1.nrt.ccgtvp5s4ca.mdawmdyynja4ztq3nwfmyjrkmtc5mdywmmqzzwjjzdrhy2i4",
      "timeCreated": "2024-05-17T06:28:24.307Z",
      "etag": "d3c3221c-b68b-4b37-9261-fb76b77c92a7",
      "freeformTags": null,
      "definedTags": null
    }
  ]
}
```

次に、_viewer_ の確認をしてみます。

```sh
kubectl -n example port-forward svc/viewer 8081:8080
```

バケット作成のリクエストを送ってみます。

```sh
curl --request POST \
  --url http://localhost:8081/os/bucket/workload-identity-2 \
  --header 'content-type/application/json' \
  -v
```

実行結果

```sh
*   Trying 127.0.0.1:8081...
* Connected to localhost (127.0.0.1) port 8081 (#0)
> POST /os/bucket/workload-identity-2 HTTP/1.1
> Host: localhost:8081
> User-Agent: curl/7.81.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 500 Internal Server Error
< Date: Fri, 17 May 2024 06:43:01 GMT
< Content-Length: 0
<
* Connection #0 to host localhost left intact
```

500 エラーが返却されました。ここで、Pod のログを確認してみると、以下のようにログ出力されています。

```sh
kubectl -n example logs -f viewer-67bc7bcc6f-fxk2h
```

実行結果

```sh
# ... 省略 ...
Error returned by ObjectStorage Service. Http Status Code: 409. Error Code: BucketAlreadyExists. Opc request id: nrt-1:A3tUNTmCXfaOh2PmXgz4-aW9UEWBHG2S2Nd4cifVLA14TUvXznhMYmovwJk61ZEl. Message: Either the bucket 'workload-identity-2' in namespace '<namespace>' already exists or you are not authorized to create it
# ... 省略 ...
```

同様にバケットの一覧も取得することができます。

```sh
curl --request GET \
  --url http://localhost:8080/os/bucket \
  --header 'content-type/application/json' | jq
```

実行結果は以下の通りです。

```json
{
  "data": [
    {
      "namespace": "<namespace>",
      "name": "workload-identity",
      "compartmentId": "<compartment-id>",
      "createdBy": "ocid1.workload.oc1.nrt.ccgtvp5s4ca.mdawmdyynja4ztq3nwfmyjrkmtc5mdywmmqzzwjjzdrhy2i4",
      "timeCreated": "2024-05-17T06:28:24.307Z",
      "etag": "d3c3221c-b68b-4b37-9261-fb76b77c92a7",
      "freeformTags": null,
      "definedTags": null
    }
  ]
}
```

# 終わりに

今回は、OKE の Workload Identity を実際に試してみました。コンテナからクラウドリソースを扱う際にも最小権限の原則を考慮することは非常に重要かと思いますので、ぜひお試しください。

# 参考

https://docs.oracle.com/ja-jp/iaas/Content/ContEng/Tasks/contenggrantingworkloadaccesstoresources.htm
