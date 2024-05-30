---
title: OCI API Gateway のレスポンスキャッシュ機能について
tags:
  - oracle
  - oci
private: false
updated_at: '2021-12-14T12:58:52+09:00'
id: c9d70733b58e5c99a8ba
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

OCI(Oracle Cloud Infrastructure) の [API Gateway](https://docs.oracle.com/ja-jp/iaas/Content/APIGateway/home.htm) から提供されている[レスポンス・キャッシュの機能](https://docs.oracle.com/ja-jp/iaas/Content/APIGateway/Tasks/apigatewayresponsecaching.htm#Response_Caching)について、ことはじめ的な記事を書きたいと思います。

# OCI API Gateway のもつレスポンスキャッシュの仕組み

OCI API Gateway は、外部のキャッシュサーバー(Redis, KeyDB, etc...)と統合することでレスポンスをキャッシュする機能を持っています。これは、パフォーマンスの向上やバックエンドサービスに対して不要な負荷をかけないという意味で重要な役割があります。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/fe7c8738-bb58-40ec-a9c1-01bf89fa4bd8.png)

全体の流れとしては、

1. クライアントから API のリクエストが発行される
2. リクエストの URL、HTTP メソッド(GET, HEAD, OPTIONS)、API デプロイメントの OCID からキャッシュ・キーを導出し、そのキャッシュ・キーを元に外部のキャッシュ・サーバーに問い合わせを行う
3. キャッシュ・ヒットの有無で以下のように振舞う
    1. キャッシュ・ヒットした場合 → キャッシュ・サーバから結果を取得し、返却する
    2. キャッシュ・ヒットしなかった場合 → 対応するバックエンドサービスから結果が返却される

といった形で機能します。実際にキャッシュ・ヒットしたかどうか？の確認は、API Gateway で HTTP Response Header に `X-Cache-Status` が追加されるので、このヘッダーをクライアント側で確認すると良いでしょう。実際に返却される `X-Cache-Status` の値は以下のようになっています。

- `X-Cache-Status: HIT`: 一致するキャッシュ・キーがキャッシュ・サーバーで見つかり、レスポンスがキャッシュ・サーバーから取得されたことを表す
- `X-Cache-Status: MISS`: 一致するキャッシュ・キーがキャッシュ・サーバーで見つからなかったため、レスポンスがバックエンドサーバーから取得されたことを表す
- `X-Cache-Status: BYPASS`: キャッシュ・サーバーに対して問い合わせが行われなかったことを表す。（通常、キャッシュ・サーバーとの通信に何か問題がある場合が多いと思います）

# API Gateway のキャッシュ機能を試してみる

## 前提

以下の前提で手順を書きたいと思います。

- Oracle Cloud Infrastructure のアカウントを有していること(NOT Always Free)
- Docker, Docker Compose がセットアップ済みの Compute Instance を有していること
- API Gateway から OCI Vault に対するポリシーの設定が完了していること
  - 参考: [Vault サービスのキャッシュ・サーバー資格証明への API ゲートウェイ・アクセス権を付与するポリシーの作成](https://docs.oracle.com/ja-jp/iaas/Content/APIGateway/Tasks/apigatewaycreatingpolicies.htm#unique_1871886445)
- Oracle Functions の開発環境の設定が完了していること（Fn CLI を用いて、Oracle Functions のアプリケーションに関数コードをデプロイすることができること）
  - 参考: [Fn Project ハンズオン](https://oracle-japan.github.io/ocitutorials/cloud-native/fn-for-beginners/)
  - 参考: [Oracle Functions ハンズオン](https://oracle-japan.github.io/ocitutorials/cloud-native/functions-for-beginners/)

## 手順

それでは、実際に構築して行きたいと思います。今回は、キャッシュ・サーバーに [Redis](https://redis.io/)、バックエンドサーバーは、[Oracle Functions](https://docs.oracle.com/ja-jp/iaas/Content/Functions/home.htm) を使用します。そのため、全体の構成は以下のようになります。

![image02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/b1675fad-5c37-5594-7aed-dc9d393c2dbe.png)

### 1. OCI Vault に Redis へアクセスするための Secret を作成する

OCI Vault に Redis へアクセスするための Secret を作成します。OCI コンソールの左上のハンバーガーメニューから、**アイデンティティとセキュリティ** > **ボールト**と選択します。

![image03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/ec468e02-5af4-6bfe-357b-9f7fc23ca844.png)

**ボールトの作成**を押し、新規にボールトを作成します。

![image04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/0b7f24e9-161c-6b82-02fe-59ec3e8e1844.png)

任意の名前を入力し、**ボールトの作成**をクリックします。

![image05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/d1441d85-830c-24d3-df25-9877ba55799e.png)

ボールトの作成が完了後、マスター暗号化キーを作成します。ボールトの詳細画面の**マスター暗号化キー**リソースで**キーの作成**をクリックします。

![image06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/9817c210-4328-0f67-ada3-318de15f7dda.png)

任意の名前を入力し、マスター暗号化キーを作成します。

![image07.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/138e94f9-f402-fd95-06ab-1fa23bf8720a.png)

次に、シークレットを作成します。ボールトの詳細画面の**シークレット**リソースで**シークレットの作成**をクリックします。

![image08.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/e70176e4-6332-6767-a2d7-6d3020c72ae4.png)

以下のように入力し、シークレットを新規に作成します。

- コンパートメントに作成: 任意のコンパートメント(API Gateway, Vault のリソースが含まれているコンパートメント)
- 名前: 任意の名前
- 説明(オプション): for oci api gateway cache feature
- 暗号化キー: 前述の手順で作成したマスター暗号化キーを指定
- シークレット・タイプ・テンプレート: プレーン・テキスト
- シークレット・コンテンツ: {"password": "\<your-password\>"}

![image09.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/96601ceb-292c-33db-32bf-be72b6b42689.png)

これで、API Gateway -> Redis へアクセスするために必要なパスワードを格納する Vault の設定は完了です。

### 2. Compute Instance に Redis を構築する

Compute に直接 Redis をインストールしても良いのですが、より手軽に実施するために今回は Docker, Docker Compose を使用して Redis を構築します。まずは、Redis 用のポートを開放します。

Compute の Firewall の設定:

```bash
sudo firewall-cmd --add-port=6379/tcp --zone=public --permanent
sudo firewall-cmd --reload
```

OCI のセキュリティ・リストの設定:

Redis を構築するインスタンスが属している VCN のセキュリティ・リストのイングレス・ルールを以下のように更新します。

- ソースタイプ: CIDR
- ソース CIDR: 0.0.0.0/0
- IP プロトコル: TCP
- ソース・ポート範囲 オプション: 未入力
- 宛先ポート範囲 オプション: 6379
- 説明 オプション: for Redis default port.

![image10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/be2f95a6-8863-8bd8-95a3-7189d714610d.png)

次に、以下の `docker-compose.yaml` を任意のディレクトリ(`$HOME/redis`等)に作成します。

```docker-compose.yaml
version: '3.8'

volumes:
  redis-data:

services:
  redis:
    image: redis:6.2.6
    container_name: redis
    ports:
      - 6379:6379
    command: redis-server --requirepass <your-password>
    volumes:
      - redis-data:/data
```

ここで、\<your-password\>は、[1. OCI Vault に Redis へアクセスするための Secret を作成する](#1-oci-vault-に-redis-へアクセスするための-secret-を作成する)で作成した Secret の値に更新してください。（本記事の例だと、`shukawam-secret`）

更新後、Redis を起動します。

```bash
docker-compose up -d
```

起動の確認

```bash
docker ps | grep redis
2e0812dd38a8        redis:6.2.6         "docker-entrypoint.s…"   6 hours ago         Up 6 hours          0.0.0.0:6379->6379/tcp   redis
```

設定したパスワードを用いてアクセス可能か確認します。

```bash
docker exec -it redis redis-cli -h <your-compute-public-ip> -p 6379
```

設定したパスワードを用いて、認証する。

```bash
auth <your-password>
OK
```

適当なキーで参照の操作が可能かどうか確認する。

```bash
get foo
(nil)
```

これで、Redis の構築は完了です。

### 3. Oracle Functions を作成する

バックエンドを構成するための Function を作成します。

```bash
fn init --runtime java11 fn-java
```

ビルド & デプロイ

```bash
cd fn-java; fn deploy --app <your-app>
```

### 4. API Gateway の設定をする

API Gateway をキャッシュ・サーバーを使用するように構築します。OCI コンソールの左上のハンバーガーメニューから、**開発者サービス** > **ゲートウェイ**と選択します。

![image11.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/955e0851-44bb-03ea-e46a-a5bd4e72b6b5.png)

**ゲートウェイの作成**をクリックし、以下のように入力して新規に API Gateway を作成します。（既に API Gateway を作成済みの場合は、キャッシュを使うように再設定が可能です）

- 名前: 任意の名前
- タイプ: パブリック
- コンパートメント: 任意のコンパートメント
- 仮想クラウド・ネットワーク: 任意の VCN
- サブネット: 任意のリージョナルサブネット
- レスポンス・キャッシングの有効化: チェックを入れる
- ホスト: 前述の手順で Redis を構築した Compute Instance のパブリック IP を入力
- ポート: 6379
- ボールト: 前述の手順で作成したボールト
- シークレット: 前述の手順で作成したシークレット
- バージョン番号: 1(更新した場合には最新のバージョンを選択する)

![image12.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/00ce60d0-8b24-eaa4-5178-dd3f1a6ec0da.png)
![image13.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/dce0517b-6e96-4f41-6c3b-ecc80292e05e.png)


次に、Oracle Functions をバックエンドに持つデプロイメントとルーティングを作成します。ゲートウェイの詳細画面で**デプロイメントの作成**をクリックします。

![image14.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/0d85f52c-c084-2ec6-a96a-31576d5de70e.png)

基本情報で以下のように入力し、**次**をクリックします。

- 名前: cache-deployment
- パス接頭辞: /cache

![image15.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/d75e011d-54df-b8a9-6726-81d5055760a9.png)

ルーティングを新規に作成するために以下のように入力し、確認画面で**作成**を押し、デプロイメントを新規に作成します。

- パス: /hello
- メソッド: GET
- タイプ: Oracle Functions
- アプリケーション: 作成済みのアプリケーション
- 機能名: fn-java
- レスポンス・キャッシング・ポリシーを表示
  - このルートのキャッシングの有効化にチェック
  - キャッシュされたレスポンスの TTL(秒): 120

![image16.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/1308859d-9aac-349b-b941-7017010fdc06.png)
![image17.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/8a0a5e4d-029c-1f74-8f0f-21383843d8e9.png)


これで、構築したキャッシュ・サーバーを使うような API Gateway とデプロイメントの設定は完了です。

### 5. キャッシュが正しく使われているかどうか確認をする

作成した API Gateway のルーティングに対してリクエストを送ってみます。

1 回目 リクエスト:

```bash
curl --dump-header - https://...apigateway.ap-tokyo-1.oci.customer-oci.com/cache/hello
```

1 回目 レスポンス:

```bash
HTTP/1.1 200 OK
Date: Fri, 05 Nov 2021 08:45:49 GMT
Content-Type: text/plain
Connection: keep-alive
Content-Length: 13
Server: Oracle API Gateway
X-Cache-Status: MISS
opc-request-id: /2C552C5FE8188F8E35BE0D97FCDEF927/5D2EDF4F87E162E086CD0E04E9B4B287
Strict-Transport-Security: max-age=31536000
X-Content-Type-Options: nosniff
X-Frame-Options: sameorigin
X-XSS-Protection: 1; mode=block

Hello, world!
```

`X-Cache-Status: MISS` でキャッシュ・ヒットしなかったことが確認できます。次に、同じリクエストを再度送ってみます。

2 回目 レスポンス:

```bash
HTTP/1.1 200 OK
Date: Fri, 05 Nov 2021 08:46:17 GMT
Content-Type: text/plain
Transfer-Encoding: chunked
Connection: keep-alive
Server: Oracle API Gateway
X-Content-Type-Options: nosniff
X-Frame-Options: sameorigin
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000
opc-request-id: /5CE6249E287877B2658F4CA18139A93B/BFBD0317FE423AB5D7DE61041B0BCF50
X-Cache-Status: HIT

Hello, world!
```

`X-Cache-Status: HIT` で確かにキャッシュ・ヒットしたことが確認できます。

# 参考

- [パフォーマンス向上のためのレスポンスのキャッシュ](https://docs.oracle.com/ja-jp/iaas/Content/APIGateway/Tasks/apigatewayresponsecaching.htm#Response_Caching)
