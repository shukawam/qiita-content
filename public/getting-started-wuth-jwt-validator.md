---
title: JWT Validatorことはじめ
tags:
  - oracle
  - oci
  - Auth0
  - oraclecloud
private: false
updated_at: '2021-12-14T13:00:48+09:00'
id: 68dd7d4172ec8b6eb16b
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

こちらの記事は [Oracle Cloud Infrastructure Advent Calendar 2020](https://qiita.com/advent-calendar/2020/oci) Day 6 の記事として書かれています。私の記事では、[OCI API Gateway](https://docs.cloud.oracle.com/ja-jp/iaas/Content/APIGateway/Concepts/apigatewayoverview.htm) から提供されている認証・認可の仕組みの仕組み(JWT Validator)について簡単な設定例を交えながら紹介したいと思います。



## API Gateway Pattern とは？（ざっくりと）

![apigateway.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/0045efb3-bcb9-afa0-9f7d-047ee6be7f79.jpeg)


※[https://microservices.io/patterns/apigateway.html](https://microservices.io/patterns/apigateway.html)から図を引用しています。

上図からも分かるように、クライアントとバックエンドの間に配置され、クライアントから見た際のバックエンドの**エントリーポイントを一つに統一**してくれています。ちなみに、このAPI Gatewayが存在しない場合、クライアントは API をコールする際にバックエンドの詳細な情報（Protocol, Host, Port etc ...）を適切に知っている必要があります。モノリシックなアプリケーションであれば気にならないかもしれませんが、マイクロサービスの場合について考えてみると、つらい事が分かります。少し前に通信していたバックエンドはもう存在していないかもしれないし、バックエンドの数が膨大だとそもそもクライアントから詳細を全て把握するのが困難な場合もあるかもしれません。そこで、クライアントからバックエンドへの通信を抽象化し管理してくれる層（API Gateway）が欲しいわけです。そういう考えの元、生まれたアーキテクチャパターンが API Gateway Pattern です。このアーキテクチャパターンを実装している代表的なサービスとしては、[Amazon API Gateway](https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/welcome.html), [Azure API Management](https://docs.microsoft.com/ja-jp/azure/api-management/), [OCI API Gateway](https://docs.cloud.oracle.com/ja-jp/iaas/Content/APIGateway/Concepts/apigatewayoverview.htm) などがあります。



# OCI API Gatewayから提供されている認証・認可の仕組み

API というものは、誰でも自由に呼び出せてしまっては困る物も存在します。（システムの利用者でない人が情報を好き勝手に取得できたりしてしまったら重要な問題に発展しますよね。）そこで、制限する必要のあるAPI に対して「誰が使えるのか」（認証）、「どの操作が許されているのか」（認可）ということを設定するのですが、こちらも各バックエンドごとに個別に設定していては非常に大変なので、複数バックエンドをまとめて管理している API Gateway に実施してもらいます。(バックエンドに対する通信は必ずAPI Gatewayを通過するため、統一的な制御が行いやすいということです) [OCI API Gateway](https://docs.cloud.oracle.com/ja-jp/iaas/Content/APIGateway/Concepts/apigatewayoverview.htm) では、この認証・認可の仕組みについて`Authorizer Functions`, `JWT Validator`という2種類の実現方法を提供しています。また、サポートしている認証・認可の機能については以下の通りです。（※2020/12/6現在）

- HTTP Basic 認証
- API Key 認証
- OAuth 認可
- Oracle Identity Cloud Service(IDCS) 認証

※より詳細な情報を知りたい場合は、こちらを参照してください。

- Authorizer Functions: [認可者機能を使用したAPIデプロイメントへの認証および認可の追加](https://docs.cloud.oracle.com/ja-jp/iaas/Content/APIGateway/Tasks/apigatewayusingauthorizerfunction.htm)
- JWT Validator: [JSON Web Token (JWT)を使用したAPIデプロイメントへの認証および認可の追加](https://docs.cloud.oracle.com/ja-jp/iaas/Content/APIGateway/Tasks/apigatewayusingjwttokens.htm)

## Authorizer Functions

簡単にまとめると、[Oracle Functions](https://docs.cloud.oracle.com/ja-jp/iaas/Content/Functions/Concepts/functionsoverview.htm) に実際の認証・認可処理を実装し特定パスのリクエストが来た際に、事前定義した Functions によって認証・認可処理が実行されるという仕組みです。この事前定義する Functions ですが、自分で実装できるからと言って好き勝手に実装してよいわけではなく、以下の仕様を満たす必要があります。

- API 呼び出し元の ID を ID プロバイダで検証するために、リクエスト属性を処理する
- API 呼び出し元が実行できる操作を決定する
- API 呼び出し元が実行できる操作を "access scope" のリストとして返却する

※2021/08/13 追記
[OCI API Gatewayの認証・認可機能について](https://qiita.com/shukawam/items/107987bba2e44222c3aa) で実装例などを取り上げました。

## JWT Validator

### JWT とは？

**J**SON **W**eb **T**oken の略称です。オープンスタンダード([RFC 7519](https://tools.ietf.org/html/rfc7519)) で、自己完結型の情報を JSON オブジェクトとして当事者間で送信する方法を定義した仕様です。ちなみに、**jot** と発音することが RFC でも推奨されています。


#### JWT(JWS形式) の構造について

JWT は、以下の情報が "."（ドット） で連結された構造となっています。(※以下で取り上げる JWT は厳密には、JWS 形式の JWT なのですが、本エントリーで JWT と記載した場合は、すべて JWS 形式の JWT の事を指します)

- Header: JWS(JSON Web Signature), JWE(JSON Web Encryption)を正しく解釈するためのメタ情報が格納されている
- Payload: 実際の情報の内容。
- Signature: Header, Payload, Secret から作成される署名のこと。アルゴリズムは、 Header で指定されたものを使用します。

これだけだと、何のことだか良く分からないと思うので実際の JWT を見てみましょう。以下のような JWT があるとします。

`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c`

注意深く見てみると、`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9` + "." + ... という 3 つのパートが "."（ドット） で連結された構造になっていることが確認できます。一つ一つのパートは、base64Url エンコードされていたり、特定のアルゴリズムで暗号化されているので一見良く分からないですが、きちんとデコードしてみると、内容が確認できます。

※以下では、Node.js を使用してHeader, Payloadをデコードしていますが、使い慣れた言語で大丈夫です。

```javascript
const base64url = require('base64url');

// base64UrlエンコードされたHeader
const encodedHeader = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9';
// base64UrlエンコードされたPayload
const encodedPayload = 'eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ';

console.log(base64url.decode(encodedHeader)); // -> {"alg":"HS256","typ":"JWT"}
console.log(base64url.decode(encodedPayload)); // -> {"sub":"1234567890","name":"John Doe","iat":1516239022}
```

ここまでデコードしてみると、 Signature がこのように作成されていることが分かります。

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  <your-256-bit-secret>)
```



### JWT Validator とは？

一言で言ってしまえば、先ほど紹介した JWT を検証する仕組みです。JWT は ID プロバイダ（Oracle Identity Cloud Service(IDCS), Auth0, Okta etc ...）によって発行されます。API の呼び出し時にクライアントから受け取った JWT を API Gateway が検証を実施し、その検証結果をもとに API の使用可否を判定します。



# 実際に作ってみた

## 全体像

### JWT Validator

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/bfc5d530-5023-3537-87dd-faac6def71af.png)

1. IDentity Provider(Auth0)からJWTを取得する
2. リクエストヘッダに`Authorization: Bearer <token(JWT)>`を含めて、APIをコールする
3. API Gatewayがリクエストに含まれているJWTの検証を行う
    1. Auth0 からJWKS(JSON Web Key Set)を取得する
    2. クライアントから送信された JWT のクレームを読み、検証を実施する
4. 検証が成功した場合は、デプロイしたFunctionsが実行される



### IDentity Provider の準備

今回は、IDentity Provider として、Auth0を使用します。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/097151a6-d21c-ba15-44b5-b377cdda93b2.png)


"CREATE API"をクリックし、以下のように情報を入力します。

- Name: API に対する名称。上記の画像のように一覧表示の際の名前として使用されます。
- Identifier: API の論理識別子。
- Signning Algorithm: Auth0 が Access Token に対して署名する際の暗号化のアルゴリズムを選択する。

これで IDentity Provider の準備は完了です。



## API Gateway からトークンの検証後に Invoke される Functions の作成

まずは、Context(開発環境)を作成します。

```bash
fn create context authorizer --provider oracle-ip
```

作成した環境に切り替えます。

```bash
fn use context authorizer
```

作成した開発環境に対して、各種設定を行います。

```bash
fn update context oracle.compartment-id <your-compartment-ocid>
fn update context api-url https://functions.<your-region>.oraclecloud.com
fn update context registry nrt.ocir.io/<your-tenancy-namespace>/<your-resistry-name>
fn update context oracle.profile <your-profile-name>
```

トークンの検証成功後に API Gateway から Invoke される関数を作成します。

```bash
fn init auth-func --runtime java11 fn-sample
```

また、アプリケーションを作成します。

![image02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/99ace25a-f9d4-97b2-ce10-3982b2c1fe26.png)


"アプリケーションの作成"の作成をクリックし、以下のように入力します。

- 名前: アプリケーションの名前。今回は、auth-sample-appとしています。
- VCN: 自分で作成したVCN
- サブネット: 自分で作成したサブネット
- タグ・ネームスペース: ご自由にどうぞ

OCI コンソールからアプリケーション作成後に、開発端末で以下のように入力し、正しい結果が返ってくればOKです！

```bash
fn list app
NAME            ID
auth-sample-app ocid1.fnapp.oc1.ap-tokyo-1.aaaaaaaaa...
```

次に作成したアプリケーションに対して、 Functions をデプロイします。

```bash
cd fn-sample
fn deploy --app auth-sample-app --verbose
```

以下の結果が返ってくれば、 Functions のデプロイは成功しています。

```bash
Deploying fn-sample to app: auth-sample-app
// ... 省略
Successfully created function: fn-sample with nrt.ocir.io/<tenancy-namespace>/<your-resistry-name>/fn-sample:0.0.4
```

試しに、実行してみると以下のようなレスポンスが返ってきます。（※FunctionsのEndpointはOCIコンソールから確認することができます。）

```bash
fn invoke --endpoint <functions-endpoint>
Hello, world!
```



## API Gateway を作成する

![image03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/9f092121-2177-3a86-ff65-dc4ae35e8c1f.png)


"ゲートウェイの作成"をクリックし、必要な情報を入力し API Gateway を作成します。

- 名前: API Gatewayの名前。今回は、auth-sample-gatewayとしています。
- タイプ: パブリック
- コンパートメント: 自分のコンパートメント
- 仮想クラウド・ネットワーク: 自分で作成したVCN
- サブネット: 自分で作成したサブネット
- タグ・ネームスペース: ご自由にどうぞ

## JWT Validator

まずは、作成した Functions を特に保護をかけずに API Gateway にデプロイしてみたいと思います。作成した API Gateway で"デプロイメント" > "デプロイメントの作成"とクリックし、API Gateway に対して、作成した Functions をデプロイします。

![image04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/9b22e28a-f5c5-f283-38a6-18beb2b03862.png)


![image05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/b91d828f-1e2c-1f8e-a137-fd57f0ac2547.png)


基本情報では以下のように入力します。

- 名前：jwt-validator-sample（お好きな名前でどうぞ）
- パス接頭辞：/jwt
- コンパートメント：自分のコンパートメント
- その他：特に入力しない

![image06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/d58c004b-38d2-e390-51c1-c59c34be1cfb.png)


ルートでは、以下のように入力します。

- パス：/hello
- メソッド：GET
- タイプ：Oracle Functions
- アプリケーション：先ほど作成したアプリケーション（auth-sample-app）
- 機能名：先ほどデプロイした Functions 名

作成が完了すると、エンドポイントが公開されるので実行してみましょう。

![image07.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/0e0fdc9e-71ae-b98a-4701-096f06a2a02f.png)


```bash
curl https://fgw5zmlsdxpqyppd4mwure4zdy.apigateway.ap-tokyo-1.oci.customer-oci.com/jwt/hello
Hello, world!
```



さて、今デプロイした API は利用者に特に制限を設けているわけではないので誰でも実行できる状態となっていますが、こちらにJWT Validatorを使用して、APIに保護をかけます。

![image08.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/a637074d-4e59-f42e-62c0-dacb150058b5.png)


作成したデプロイメントの編集で"APIリクエスト・ポリシー" > "認証" > "追加"とクリックします。

![image09.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/b4a5e670-6611-73b8-2a40-28fa31eb8338.png)

![image10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/74344446-7978-022a-4092-db53f95aa886.png)


それぞれ以下のように入力し、変更を適用します。

- 認証タイプ：JWT
- 認証トークン：ヘッダー
- ヘッダー名：Authorization
- 認証スキーム：Bearer
- 発行者：`https://<your-tenant-name>.auth0.com`
- オーディエンス：`https://oci-jwt-validator`（作成したAuth0 APIの論理識別子）
- 公開キー
  - タイプ：リモートJWKS
  - URI：`https://<your-tenant-name>.auth0.com/.well-known/jwks.json`
  - 最大キャッシュ期間(時間)：1

更新が完了したら、先ほどのエンドポイントに対して再度アクセスをしてみると、`HTTP 401`が返却されることが確認できると思います。

```bash
curl https://fgw5zmlsdxpqyppd4mwure4zdy.apigateway.ap-tokyo-1.oci.customer-oci.com/jwt/hello
{"message":"Unauthorized","code":401}
```

サンプルのアクセストークンが提供されているので、こちらを使用して API を実行してみましょう。

![image11.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/0cd328c9-7b82-7992-1266-984173b8c7e3.png)


```bash
curl https://fgw5zmlsdxpqyppd4mwure4zdy.apigateway.ap-tokyo-1.oci.customer-oci.com/jwt/hello \
--header 'authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6Ik1vR25OR1FEWGw0dlpBdFNCc3FISiJ9.eyJpc3MiOiJodHRwczovL3NodWthd2FtLW9yYS51cy5hdXRoMC5jb20vIiwic3ViIjoiM2ZqV3UwQklPY0s4SEJLSzQ0Vmc3cUppSlIxdjFNd2pAY2xpZW50cyIsImF1ZCI6Imh0dHBzOi8vb2NpLWp3dC12YWxpZGF0b3IiLCJpYXQiOjE2MDU5MzQwNzQsImV4cCI6MTYwNjAyMDQ3NCwiYXpwIjoiM2ZqV3UwQklPY0s4SEJLSzQ0Vmc3cUppSlIxdjFNd2oiLCJndHkiOiJjbGllbnQtY3JlZGVudGlhbHMifQ.Y0UDrDrCU9jlDnRqkNPAB2GWdCs9k_UoRCsqYcL0ZrXjLCRO42cSPR3d3q8t0bhF8rOxMYiMtd0VQK2xBJGycmT9sm0CxijB11DalWgyPgDCqfzr-_24QAT5zgLV7NHreGh7YRkzutOMxg6l3cU8rwmZrwFpAM6gWnuuqQdvxzD-PLie9PsGul9VzYD5jSIs9-HPDZ6PEmb0DYaHIotpPFO9iltszEtysHTBccPE9pOPcCK1F7wb3m-sou2zA2xo2DIqfsXR8E_tFs2yJ5PSCgjDNWHaHc0TipysEc5AeP85-ubanmFcH1E6QJPCakYR9nsx0iTF54RGDc5xVQRBMw'
Hello, world!
```

こちらで、API Gateway にデプロイした API に対してアクセストークンによる保護をかけることができました。スコープの設定によって細かな認可処理等も設定できるので是非遊んでみてください！



# 終わりに

今回は、本当に簡単なサンプルのようなものを作成したので特に意識していませんが、実際に使う際にはフロントエンドまで含めてトークンの管理場所などのライフサイクル等も検討しないといけないですね。



# 参考

- [https://microservices.io/patterns/apigateway.html](https://microservices.io/patterns/apigateway.html)
- [OCI API Gateway（英語）](https://docs.cloud.oracle.com/en-us/iaas/Content/APIGateway/Concepts/apigatewayoverview.htm)
- [https://jwt.io/introduction/](https://jwt.io/introduction/)
