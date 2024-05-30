---
title: Helidon 2.4.0 で導入された OpenID Connect のログアウト機能(RP-Initiated Logout)について
tags:
  - Java
  - oci
  - microprofile
  - Helidon
private: false
updated_at: '2021-12-14T12:57:28+09:00'
id: 130889260948bf3dc7c6
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

こちらの記事は、[Oracle Cloud Infrastructure Advent Calendar 2021](https://qiita.com/advent-calendar/2021/oci) Day 6 の記事として書かれています。
11 月の上旬ごろ、Helidon のバージョン 2.4.0 がリリースされました。今回の記事はアップデートされた内容の中から（個人的に待望していた） OpenID Connect 1.0 のログアウト機能(RP-Initiated Logout)について解説したいと思います。

# Helidon が提供していた OIDC の機能

一般的な OpenID Connect 1.0 の認可コードフローは以下のようになっています。ここで、下図の赤線・文字で表現されている箇所が Helidon の Security Provider(OIDCProvider) が提供している箇所となります。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/7834a954-581c-02de-db27-ad499da9dee1.png)

いわゆる Relying Party の実装を提供する機能なので、

- OpenID Provider（以下、OP） の認可エンドポイントに対して、認証リクエストを投げる
- OP が発行した認可コードと ID・アクセストークンを引き換えるために、トークンエンドポイントにリクエストを投げる
- OP から発行された ID トークンの検証を行う

といったことを設定 + ちょっとした実装で可能にしてくれるものです。尚、認証後の API 実行に使用するアクセストークンはデフォルトでは Cookie に保存するような実装となっているので、ログアウトを実現したい場合は Cookie を削除する & OP のログアウト API を実行するということを別途自分で実装する必要がありました。

# Helidon 2.4.0 でのアップデート

Helidon 2.4.0 では、上記の Relying Party の実装に加えて、ログアウト用の機能が追加されています。（PR は[こちら](https://github.com/oracle/helidon/pull/3456)）これによって、新しく追加されたエンドポイント(`/oidc/logout`)を叩くと、裏側では Cookie の削除 & OP のログアウト API を叩くということが自動的に行われます。OpenID Connect でいうところの [RP-Initiated Logout](https://openid.net/specs/openid-connect-rpinitiated-1_0.html) を実現するための実装です。典型的には、以下のパラメータ群を RP から OP へ送ることでログアウトを実現します。（IDCS[^1] では `id_token_hint`, `post_logout_redirect_uri`, `state` をサポートしています。）

[^1]: Oracle Identity Cloud Service

| パラメータ               | 必須 | 説明                                                                                                                  |
| ------------------------ | ---- | --------------------------------------------------------------------------------------------------------------------- |
| id_token_hint            | 必須 | OP から RP へ発行された ID トークン                                                                                   |
| post_logout_redirect_uri | 任意 | OP でのログアウト処理完了後のリダイレクト先を指定する                                                                 |
| state                    | 任意 | ログアウトの要求、post_logout_redirect_uri で指定したエンドポイントへのコールバック間の状態を維持するためのパラメータ |
| ui_locales               | 任意 | ロケールを指定する                                                                                                    |

ちなみに、今回のアップデートからログアウトの機能を有効化した場合は、デフォルトでは Cookie に ID トークンを保存するような動きとなるため Encrypted JWT もフレームワークとしてサポートされるようになりました。

# 一通り実装してみる

それでは、実際にログイン ~ ログアウトまでを一通り実装していきたいと思います。今回は、以下のような構成で作っています。

- OP: IDCS
  - Keycloak や、Auth0 でもおそらく代用可能（未確認）なので、普段使い慣れているもので是非お試しください
- Relying Party: Helidon MP

## エンドポイント

今回登場するエンドポイントは以下のようになっています。

| エンドポイント | 実装    | 概要                                                                            |
| -------------- | ------- | ------------------------------------------------------------------------------- |
| /auth/login    | 自分    | JAX-RS で実装する OIDC ログインのエントリーポイント                             |
| /oidc/redirect | Helidon | Helidon Security Provider が提供する OIDC のリダイレクト URL                    |
| /oidc/logout   | Helidon | Helidon Security Provider が提供するログアウトのエントリーポイント              |
| /auth/logout   | 自分    | JAX-RS で実装する OIDC ログアウト後のリダイレクト先（post_logout_redirect_uri） |

これをログインとログアウトのフローに当てはめると以下のようになります。

**ログイン**:

![image02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/cacaf2cc-4f1e-f352-af77-7dc16b9ab2a8.png)

**ログアウト**:

![image03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/b3922abc-6f91-5e75-babc-37f14b3a178e.png)

## OP(IDCS) の準備

アプリケーションを新規に作成します。ログイン後のダッシュボード画面左上のハンバーガーメニューを押し、**アプリケーション**を選択します。

![image04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/1e316a11-056d-d172-3404-5a23c3a472ed.png)

**追加**を押します。

![image05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/8e81dcc0-867b-33c2-dcd8-c8eaeb1fd868.png)

**機密アプリケーション**を選択します。（OAuth, OpenID Connect でいうところの Confidential Client に相当します）

![image06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/0da0a425-b6db-1aa9-6b92-499329238967.png)

以下のように入力し、**次**をクリックします。

- 名前: Helidon OIDC OP

![image07.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/030c5af3-700c-032b-c5a5-9b00f2d36ee4.png)

OP としての振舞いを設定するために、以下のように入力し、**次**をクリックします。

- **このアプリケーションをクライアントとして今すぐ構成します**にチェック
- 許可される権限付与タイプ: クライアント資格証明、JWT アサーション、認可コード
- **HTTPS 以外の URL を許可**にチェック
- リダイレクト URL: `http://localhost:8080/oidc/redirect`
- ログアウト後のリダイレクト URL: `http://localhost:8080/auth/logout`

![image08.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/70b2ab15-daa4-1fd6-be54-89bb0bc8a046.png)

デフォルトのまま**次**をクリックする。

![image09.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/81d6b979-0ae9-7725-aab6-56d15450f80f.png)

デフォルトのまま**次**をクリックする。

![image10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/e6ee46a1-d567-70e2-0016-550370bd869a.png)

デフォルトのまま**次**をクリックする。

![image11.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/042e213f-b6c9-d31e-d718-bba6bd7a8dc8.png)

これで、アプリケーションが作成できたので**アクティブ化**します。

![image12.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/f66beb8a-6940-1b8a-928c-246f3a85f997.png)

次に、認証対象であるエンドユーザーを登録します。ユーザータブから**ユーザーの割当て**を選択し、現在操作しているユーザーを選択します。（これ用に新しく作成しても結構です）

![image13.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/901f4b5a-67ee-b590-8b7d-a6577f6e3d74.png)

RP の実装で使用するため、クライアント ID とクライアント・シークレットはどこかに控えておきます。

![image14.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/e2502501-3b43-9d0f-b2c8-ee21d25854dd.png)

これで、OP の設定は完了です！

## Relying Party の実装

まずは、Helidon CLI を用いて、アプリケーションのひな形を生成します。`groupid`, `package`等は、好きなように設定してください。

```bash
heldion init \
--flavor MP \
--build MAVEN \
--version 2.4.0 \
--archtype bare \
--groupid me.shukawam \
--artifactid helidon-oidc-sample \
--package me.shukawam.helidon \
--name helidon-oidc-sample
```

OIDC 関連の Security Provider を依存関係に含めるために以下の記述を pom.xml に追記します。

```pom.xml
<dependencies>
  <!-- ... -->
  <dependency>
      <groupId>io.helidon.microprofile</groupId>
      <artifactId>helidon-microprofile-oidc</artifactId>
  </dependency>
  <dependency>
      <groupId>io.helidon.security.providers</groupId>
      <artifactId>helidon-security-providers-idcs-mapper</artifactId>
  </dependency>
  <!-- ... -->
</dependencies>
```

次に、OP の認可エンドポイントを叩く際やトークンリクエスト時の Basic 認証に使用する Client ID/Secret 等を設定ファイル(MicroProfile Config)に定義するために、resource 直下に application.yaml を新規に作成します。（※resources/META-INF/microprofile-config.properties に追記しても良いです。アプリケーション中に両方含まれている場合は、application.yaml が優先されます。）

```application.yaml
# Microprofile server properties
server:
  port: 8080
  host: 0.0.0.0

# Security config
security:
  # Set to true for production - if set to true, clear text passwords will cause failure
  config:
    require-encryption: false
  properties:
    # Identity Provider - IDCS
    idcs-uri: <your-idcs-tenant-uri>
    idcs-client-id: <your-idcs-client-id>
    idcs-client-secret: <your-idcs-client-secret>
    frontend-uri: <your-front-end>
    audience: <your-audience>
  providers:
    - abac:
      # Adds ABAC Provider - it does not require any configuration
    - oidc:
        client-id: ${security.properties.idcs-client-id}
        client-secret: ${security.properties.idcs-client-secret}
        identity-uri: ${security.properties.idcs-uri}
        proxy-host: ${security.properties.proxy-host}
        frontend-uri: ${security.properties.frontend-uri}
        audience: ${security.properties.audience}
        server-type: idcs
        idcs-roles: true
        # OIDC Logout support
        logout-enabled: true
        post-logout-uri: auth/logout
```

変数として定義している箇所（`idcs-uri`, `idcs-client-id`, `idcs-client-secret`, `frontend-uri`, `audience`）を自分の設定に合わせて変更します。もちろん、環境変数から取得したり、システムプロパティとして設定しても良いです。（本来はそちらの方が望ましい）

次に、ログインの入り口、ログアウトのリダイレクト先となるエンドポイントを以下のように実装します。

```AuthResource.java
import io.helidon.security.SecurityContext;
import io.helidon.security.annotations.Authenticated;

import javax.json.Json;
import javax.json.JsonBuilderFactory;
import javax.json.JsonObject;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.Context;
import java.util.Collections;

@Path("auth")
public class AuthResource {
    private static final JsonBuilderFactory JSON = Json.createBuilderFactory(Collections.emptyMap());

    @GET
    @Path("login")
    @Produces("application/json")
    @Authenticated // ... 1
    public JsonObject login(@Context SecurityContext securityContext) { // ... 2
        return JSON.createObjectBuilder()
                .add("message", "Login Success!")
                .add("user", securityContext.userName()) // ... 3
                .build();
    }

    @GET
    @Path("logout") // ... 4
    @Produces("application/json")
    public JsonObject logout() {
        return JSON.createObjectBuilder()
                .add("message", "Logout Success!")
                .build();
    }
}
```

簡単に補足しておきます。

1. ログインのエントリーポイントとなるメソッドに、`@Authenticated` を付けると、Config で設定した Security Provider(今回の例だと、[io.helidon.security.providers.oidc.OidcProvider](https://oracle-japan-oss-docs.github.io/helidon/docs/v2/apidocs/io.helidon.security.providers.oidc/io/helidon/security/providers/oidc/OidcProvider.html))によって、エンドユーザーの認証処理が実行される
2. SecurityContext(ユーザーに関するセキュリティ情報が格納されているコンテキスト)を受け取る
3. SecurityContext からユーザー名を取得し、クライアントへ返却する
4. `/oidc/logout` 実行後、OP からリダイレクトされるエンドポイント(post_logout_redirect_uri)の定義

実装は、以上で完了です。簡単！

## 簡易的に動作確認

OP の設定、Relying Party の実装が完了したので簡易的に動作確認をします。アプリケーションを起動後、ブラウザで`http://localhost:8080/auth/login`にアクセスすると、IDCS から提供されているログイン画面にリダイレクトされるので、資格情報を入力し、ログインします。

![image15.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/db417329-5574-3a48-c0a2-f648dd7da36e.png)

ログインが完了すると、以下のような JSON が返却されます。

```json
{
  "message": "Login Success!",
  "user": "<your-username>"
}
```

この状態で、再度ログイン用のエンドポイント(`http://localhost:8080/auth/login`)を叩いてみると、認証画面を介さずに以下のような結果が返ってくることが確認できます。

```json
{
  "message": "Login Success!",
  "user": "<your-username>"
}
```

続いて、ログアウトをしてみます。ブラウザで`http://localhost:8080/oidc/logout`にアクセスし、ログアウトが完了すると以下のような JSON が返却されます。

```json
{
  "message": "Logout Success!"
}
```

この状態で再度、ログイン用のエンドポイントエンドポイント(`http://localhost:8080/auth/login`)を叩いてみると、きちんと認証画面へリダイレクトされることが確認できます。

# 終わりに

ログアウトに関してもフレームワークとして機能が提供されるようになったのは個人的に非常に嬉しいです。
実装したサンプルコードの全量は、こちらの[リポジトリ](https://github.com/shukawam/helidon-oidc-sample)を参照ください。

# 宣伝

実は、Helidon には[日本語ドキュメント](https://oracle-japan-oss-docs.github.io/helidon/docs/v2/#/about/01_overview)も存在します。英語だから取っつき辛かったという方もこの機会に是非触ってみてください！

# 参考

- [Helidon 日本語ドキュメント](https://oracle-japan-oss-docs.github.io/helidon/docs/v2/#/about/01_overview)
- [Helidon Pull Request - OIDC logout](https://github.com/oracle/helidon/pull/3456)
