---
title: 'Helidon MP, WebAuthn4Jで簡易的なRelying Partyを実装してみる'
tags:
  - WebAuthn
  - FIDO2
  - Helidon
  - WebAuthn4J
private: false
updated_at: '2021-12-14T13:00:17+09:00'
id: 4c1625bb6ae00e6b17f1
organization_url_name: oracle
slide: false
ignorePublish: false
---
# 始めに

Java で WebAuthn の Relying Party を実装してみました。



# 構成

作ったサンプルコードは[リポジトリ](https://github.com/shukawam/fido2-example)に格納してあります。リポジトリルートの`docker-compose.yml`を使用すると、以下の構成が起動されます。

![architecture.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/1ee5d6dc-da79-cf9d-314b-452779e2ce92.png)


構成図にも記されていますが、こんな技術スタックで作ってみました。

- Relying Party
  - Java
        - [Helidon MP](https://helidon.io/#/)
            - Oracle主導で開発している軽量フレームワークで、今回はREST APIの実装用途で使っています
        - [WebAuthn4J](https://github.com/webauthn4j/webauthn4j)
            - Attestation, Assertionの検証用に使用
            - 外部ライブラリへの依存を極力抑えたような思想で、導入のハードルがかなり低そうだったので選定しました
- WebAuthn Client
  - [Angular](https://angular.jp/)



# 実装時のポイント

ここから実装時のポイントをかいつまんで紹介していきます。さすがに全量は紹介できないので細かい所を確認したい場合は、[リポジトリ](https://github.com/shukawam/fido2-example)をご参照ください。

また、今回は認証器としてYubikeyを使用しています。持っていない場合は、Chromeの拡張に[Virtual Authenticators Tab](https://chrome.google.com/webstore/detail/virtual-authenticators-ta/gafbpmlmeiikmhkhiapjlfjgdioafmja)があるのでこれを使うと良いでしょう。



## 登録フロー

![regist.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/60d100de-936a-2f98-43b6-c245ada17ddb.png)




<p style="text:center">出典: <a href="https://www.w3.org/TR/webauthn-1/images/webauthn-registration-flow-01.svg">https://www.w3.org/TR/webauthn-1/images/webauthn-registration-flow-01.svg</a></p>

登録のフローはざっと以下のようになっています。

1. ユーザーの登録要求を行う
2. `navigator.credentials.create()`の実行に必要なパラメータをRPで生成し、クライアント(WebAuthn Client)に返却する
3. RPから取得した情報を元に WebAuthn - `navigator.credentials.create()`を実行する
4. 認証器で`authenticatorMakeCredential()`が実行され、秘密鍵・公開鍵の鍵ペアの作成、Attestation[^1] が生成される
5. 認証器で生成された情報がクライアント(WebAuthn Client)へ返却される
6. 認証器から取得した情報のうち、検証に必要な情報(clientDataJSON, attestationObject)をRPへ対して送信する
7. RPで認証器で生成された情報の検証を行う

[^1]: Credential ID(資格情報を一意に識別するID)や認証時に使用する公開鍵等を含んだ情報



このうち、4, 5に関しては今回の対象外とさせてもらいます。[別記事](https://shukawam.github.io/blog/blog/2021/0324-webauthn-deep-dive/)にて少し深掘りしてみたので興味があれば読んでみてください。また、通常4, 5に関してはWebAuthn + CTAPで規定されており、各対応ブラウザ(+ 認証器)が良しなに処理してくれるので、実は実装上あまり意識しなくてもよい箇所ではあります。



### 1. ユーザーの登録要求

プロトコル、フォーマット等は決まっていないので自分で決める必要があります。今回は、`GET /webauthn/attestation/options/{email}`でリクエストを発行しています。

```typescript
// http://localhost:8081/webauthn/attestation/options/{email}
this.httpClient.get<AttestationServerOptions>(`${this.ATTESTATION_OPTION}/${email}`, {
    headers: {
        'Content-type': 'application/json',
    },
}).toPromise()
```



### 2. `navigator.credentials.create()`実行に必要なパラメータの生成

`navigator.credentials.create()`の実行に必要なパラメータを生成します。この時、必ず必要となるパラメータは

- challenge: リプライ攻撃防止用のパラメータで16バイト以上のランダムバッファーである必要がある
- pubKeyCredParams: RPが受け入れ可能なCredentialのタイプ(`public-key`固定)とアルゴリズムを指定する
  - alg: Credentialのアルゴリズムを指定する
  - type: `public-key`固定
- rp: Relying Partyの情報
  - id: RPを一意に識別するIDで、有効なドメインを指定する
  - name: RPの名称
  - icon: アイコンをURL形式で指定する
- user: ユーザーの情報
  - id: ユーザーを一意に識別するID
  - name: ユーザーの名称(入力用)
  - displayName: ユーザーの名称(表示用)
  - icon: ユーザーのアイコンをURL形式で指定する

の4つなので今回は以下のように実装してみました。

```java
var challenge = new DefaultChallenge(); // ... 1
var user = entityManager.find(Users.class, email);
if (user == null) {
    user = createNewUser(email, challenge); // ... 2
} else {
    // do something.
}
// Require
var userInfo = new PublicKeyCredentialUserEntity(
    user.getId(), // id
    user.getEmail(), // username(email)
    user.getEmail() // displayName
);
var pubKeyCredParams = Arrays.asList(
    new PublicKeyCredentialParameters(
        PublicKeyCredentialType.PUBLIC_KEY,
        COSEAlgorithmIdentifier.ES256),
    new PublicKeyCredentialParameters(
        PublicKeyCredentialType.PUBLIC_KEY,
        COSEAlgorithmIdentifier.RS256)
);
// Optional
var excludeCredentials = entityManager
    .createNamedQuery("getCredentialById", Credentials.class)
    .setParameter("credentialId", user.getCredentialId())
    .getResultStream()
    .map(credential -> new PublicKeyCredentialDescriptor(
        PublicKeyCredentialType.PUBLIC_KEY,
        Base64UrlUtil.decode(credential.getCredentialId()),
        Collections.emptySet())
        ).collect(Collectors.toList());
var authenticatorSelectionCriteria = new AuthenticatorSelectionCriteria(
    AuthenticatorAttachment.CROSS_PLATFORM,
    false,
    UserVerificationRequirement.PREFERRED
);
var publicKeyCredentialCreationOptions = new PublicKeyCredentialCreationOptions(
    rp,
    userInfo,
    challenge,
    pubKeyCredParams,
    TimeUnit.SECONDS.toMillis(6000),
    excludeCredentials,
    authenticatorSelectionCriteria,
    AttestationConveyancePreference.DIRECT,
    null
);
return publicKeyCredentialCreationOptions; // ... 3
```



#### 1. challengeの生成

[WebAuthn4J](https://github.com/webauthn4j/webauthn4j)では、`com.webauthn4j.data.client.challenge.Challenge`を自分で実装するか、`com.webauthn4j.data.client.challenge.DefaultChallenge`が用意されているのでそのどちらかを使うことになりますが、今回は`com.webauthn4j.data.client.challenge.DefaultChallenge`を使用しています。(ランダムに生成したUUIDからバイト配列を生成しているようです)



#### 2.  ユーザー情報(challenge)を保存しておく

登録処理の終了時に認証器から生成された情報内に含まれるchallengeとRPで生成したchallengeが一致するかどうかの検証が必要なので何らかの手段で保存しておきます。今回は、Databaseに保存していますがHTTP Sessionなどでもよいでしょう。



#### 3. PublicKeyCredentialCreationOptionsを返却する

クライアントに返却する際に、challengeなどバイト配列のプロパティはBase64urlエンコードしてから返却するので、そのためのクラスを生成します。今回はこんな形で実装しました。

```AttestationServerOptions.java
import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonProperty;
import com.webauthn4j.data.*;
import com.webauthn4j.data.extension.client.AuthenticationExtensionsClientInputs;
import com.webauthn4j.data.extension.client.RegistrationExtensionClientInput;

import java.util.List;

public class AttestationServerOptions {
    public PublicKeyCredentialRpEntity rp;
    public MyPublicKeyCredentialUserEntity user;
    public String challenge;
    public List<MyPublicKeyCredentialParameters> pubKeyCredParams;
    public Long timeout;
    public List<MyPublicKeyCredentialDescriptor> excludeCredentials;
    public MyAuthenticatorSelectionCriteria authenticatorSelection;
    public String attestation;
    public AuthenticationExtensionsClientInputs<RegistrationExtensionClientInput> extensions;

    @JsonCreator
    public AttestationServerOptions(@JsonProperty("rp") PublicKeyCredentialRpEntity rp, @JsonProperty("user") MyPublicKeyCredentialUserEntity user, @JsonProperty("challenge") String challenge, @JsonProperty("pubKeyCredParams") List<MyPublicKeyCredentialParameters> pubKeyCredParams, @JsonProperty("timeout") Long timeout, @JsonProperty("excludeCredentials") List<MyPublicKeyCredentialDescriptor> excludeCredentials, @JsonProperty("authenticatorSelection") MyAuthenticatorSelectionCriteria authenticatorSelection, @JsonProperty("attestation") String attestation, @JsonProperty("extensions") AuthenticationExtensionsClientInputs<RegistrationExtensionClientInput> extensions) {
        this.rp = rp;
        this.user = user;
        this.challenge = challenge;
        this.pubKeyCredParams = pubKeyCredParams;
        this.timeout = timeout;
        this.excludeCredentials = excludeCredentials;
        this.authenticatorSelection = authenticatorSelection;
        this.attestation = attestation;
        this.extensions = extensions;
    }
}

```

最後に、生成した`PublicKeyCredentialCreationOptions`からレスポンス用のオブジェクトに詰め替えてクライアントに返却します。お好きな方法でどうぞ。参考までに。

```java
@GET
@Path("attestation/options/{email}")
@Produces(MediaType.APPLICATION_JSON)
public AttestationServerOptions attestationOptions(@PathParam("email") String email) {
    var publicKeyCredentialCreationOptions = webAuthnService.createServerOptions(email);
    return new AttestationServerOptions(
        publicKeyCredentialCreationOptions.getRp(),
        new MyPublicKeyCredentialUserEntity(
            Base64UrlUtil.encodeToString(publicKeyCredentialCreationOptions.getUser().getId()),
            publicKeyCredentialCreationOptions.getUser().getName(),
            publicKeyCredentialCreationOptions.getUser().getDisplayName()
        ),
        Base64UrlUtil.encodeToString(publicKeyCredentialCreationOptions.getChallenge().getValue()),
        publicKeyCredentialCreationOptions.getPubKeyCredParams().stream().map(publicKeyCredentialParameters -> new MyPublicKeyCredentialParameters(
            publicKeyCredentialParameters.getType().getValue(),
            publicKeyCredentialParameters.getAlg().getValue()
        )).collect(Collectors.toList()),
        publicKeyCredentialCreationOptions.getTimeout(),
        publicKeyCredentialCreationOptions.getExcludeCredentials().stream().map(publicKeyCredentialDescriptor -> new MyPublicKeyCredentialDescriptor(
            publicKeyCredentialDescriptor.getType().getValue(),
            Base64UrlUtil.encodeToString(publicKeyCredentialDescriptor.getId()),
            publicKeyCredentialDescriptor.getTransports()
        )).collect(Collectors.toList()),
        new MyAuthenticatorSelectionCriteria(
            publicKeyCredentialCreationOptions.getAuthenticatorSelection().getAuthenticatorAttachment().getValue(),
            publicKeyCredentialCreationOptions.getAuthenticatorSelection().isRequireResidentKey(),
            publicKeyCredentialCreationOptions.getAuthenticatorSelection().getUserVerification().getValue()
        ),
        publicKeyCredentialCreationOptions.getAttestation().getValue(),
        publicKeyCredentialCreationOptions.getExtensions()
    );
}
```





### 3. `navigator.credentials.create()`を実行する

RPから取得した情報を元に`navigator.credentials.create()`の実行に必要なパラメータを組み立てて実行します。具体的にはこんな形で。

```typescript
  public async createCredential(email: string): Promise<Credential | null> {
    return this.fetchAttestationOptions(email).then((fetchOptions) => {
      let credentialCreationOptions: CredentialCreationOptions = {
        publicKey: fetchOptions,
      };
      return navigator.credentials.create(credentialCreationOptions);
    });
  }

  private async fetchAttestationOptions(
    email: string
  ): Promise<PublicKeyCredentialCreationOptions> {
    return this.httpClient
      .get<AttestationServerOptions>(`${this.ATTESTATION_OPTION}/${email}`, {
		headers: {
          'Content-type': 'application/json',
        },
      })
      .toPromise()
      .then((serverOptions) => {
        console.log('serverOptions', serverOptions);
        return {
          // Require
          rp: serverOptions.rp,
          user: {
            id: Base64urlUtil.base64urlToArrayBuffer(serverOptions.user.id),
            name: serverOptions.user.name,
            displayName: serverOptions.user.displayName,
          },
          challenge: Base64urlUtil.base64urlToArrayBuffer(
            serverOptions.challenge
          ),
          pubKeyCredParams: serverOptions.pubKeyCredParams,
          // Optionally
          timeout: serverOptions.timeout,
          excludeCredentials: serverOptions.excludeCredentials.map(
            (credential) => {
              return {
                type: credential.type,
                id: Base64urlUtil.base64urlToArrayBuffer(credential.id),
                transports: credential.transports,
              };
            }
          ),
          authenticatorSelection: serverOptions.authenticatorSelection,
          attestation: serverOptions.attestation,
          extensions: serverOptions.extensions,
        };
      });
  }
```

そんなにポイントはないですが、RP側でBase64urlエンコードしてから返却された項目はしっかりとデコードします。(challengeやuser.idなど)



### 6. 認証器が生成した情報から検証に必要な情報をRPへ送信する

認証器の`authenticatorMakeCredential()`が実行されると最終的に以下のようなデータが戻ってきます。

```json
{
  "id": "xTzphZPuJyfW12TAT…",
  "rawId": ArrayBuffer(64) {},
  "response": {
    "attestationObject": ArrayBuffer(1024),
    "clientDataJSON": ArrayBuffer(116) {}
  },
  "type": "public-key"
}
```

それぞれのプロパティを見ていきましょう。

| パラメータ名      | 概要                                                         |
| ----------------- | ------------------------------------------------------------ |
| id                | rawId を base64url エンコードしたもの                        |
| rawId             | Credential 毎に一意に定められている                          |
| attestationObject | 認証用の公開鍵や CredentialID、署名などが含まれている        |
| clientDataJSON    | challenge, oritin, type などが含まれている clientData を JSON シリアライズしたもの |
| type              | `public-key`固定                                             |

このうち、`attestationObject`, `clientDataJSON`を Relying Party へ送信し検証が成功すれば登録処理は完了です。

```typescript
// http://localhost:8081/webauthn/attestation/result
this.httpClient.post<AttestationResult>(`${this.ATTESTATION_RESULT}`,
	{
    	email: email,
    	attestationObject: Base64urlUtil.arrayBufferToBase64url(
            authenticatorAttestationResponse.attestationObject
        ),
    	clientDataJSON: Base64urlUtil.arrayBufferToBase64url(
            authenticatorAttestationResponse.clientDataJSON
        ),
	}).toPromise();
```



### 7. RPで認証器で生成された情報の検証を行う

認証器で生成された情報の検証は

- challengeがRPで生成されたものと一致するか
- originが期待通りか
- clientDataHashの署名と認証器用の証明書チェーンを使ってattestationを検証結果が正しいか

ということを行いますが、`com.webauthn4j.WebAuthnManager#validate(com.webauthn4j.data.RegistrationData, com.webauthn4j.data.RegistrationParameters)`というメソッドでこれらの検証を実施してくれるのでそのために必要なパラメータを組み立てます。一連の流れは以下の通り。

```java
var origin = Origin.create("http://localhost");
var user = entityManager.find(Users.class, email);
// 保存しておいたchallengeを取得する
var challenge = new DefaultChallenge(user.getChallenge());
var serverProperty = new ServerProperty(origin, rp.getId(), challenge, null);
var registrationRequest = new RegistrationRequest(attestationObject, clientDataJSON);
var registrationParameters = new RegistrationParameters(serverProperty, true);
RegistrationData registrationData;
try {
    registrationData = WebAuthnManager.createNonStrictWebAuthnManager().parse(registrationRequest);
} catch (DataConversionException e) {
    // do something.
}
// attestation validation
try {
    // 認証器が生成した情報の検証
    WebAuthnManager.createNonStrictWebAuthnManager().validate(registrationData, registrationParameters);
} catch (ValidationException e) {
    // do something
}
// persist authenticator object, which will be used in authentication process.
var authenticator = new AuthenticatorImpl(
    registrationData.getAttestationObject().getAuthenticatorData().getAttestedCredentialData(),
    registrationData.getAttestationObject().getAttestationStatement(),
    registrationData.getAttestationObject().getAuthenticatorData().getSignCount()
);
var credentialId = registrationData.getAttestationObject().getAuthenticatorData().getAttestedCredentialData().getCredentialId();
// store credential to user table
user.setCredentialId(Base64UrlUtil.encodeToString(credentialId));
logger.info(user.getCredentialId());
// store authenticator
persistAuthenticator(credentialId, authenticator);
```

また、最後にその後の認証処理で使用するので認証器の情報を永続化しておきます。その際にシリアライズ／デシリアライズするためのクラスも用意されているのでありがたく使わせてもらいます。

```java
    private void persistAuthenticator(byte[] credentialId, Authenticator authenticator) {
        // serialize authenticator
        var objectConverter = new ObjectConverter();
        var attestedCredentialDataConverter = new AttestedCredentialDataConverter(objectConverter);
        var serializedAttestedCredentialData = attestedCredentialDataConverter.convert(authenticator.getAttestedCredentialData());
        var attestationStatementEnvelope = new AttestationStatementEnvelope(authenticator.getAttestationStatement());
        var serializedEnvelope = objectConverter.getCborConverter().writeValueAsBytes(attestationStatementEnvelope);
        var serializedTransports = objectConverter.getJsonConverter().writeValueAsString(authenticator.getTransports());
        var serializedAuthenticatorExtensions = objectConverter.getCborConverter().writeValueAsBytes(authenticator.getAuthenticatorExtensions());
        var serializedClientExtensions = objectConverter.getJsonConverter().writeValueAsString(authenticator.getClientExtensions());
        entityManager.persist(new Credentials(
                Base64UrlUtil.encodeToString(credentialId),
                serializedAttestedCredentialData,
                serializedEnvelope,
                serializedTransports,
                serializedAuthenticatorExtensions,
                serializedClientExtensions,
                authenticator.getCounter()
        ));
    }
```

特に例外(`com.webauthn4j.converter.exception.DataConversionException` `com.webauthn4j.validator.exception.ValidationException`)が発生したり、DBへの永続化が失敗しなければクライアントに対して登録処理が完了したことを通知します。



## 認証フロー


![auth.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/44e28ce6-7784-5956-7bdb-a542a9f70a4f.png)


<p style="text:center">出典: <a href="https://www.w3.org/TR/webauthn-1/images/webauthn-authentication-flow-01.svg">https://www.w3.org/TR/webauthn-1/images/webauthn-authentication-flow-01.svg</a></p>

認証のフローはざっと以下のようになっています。(登録フローとかなり似通っているので登録フローが実装できれば認証フローは結構楽にできると思います。)

1. ユーザーの認証要求を行う
2. `navigator.credentials.get()`の実行に必要なパラメータをRPで生成し、クライアント(WebAuthn Client)に返却する
3. RPから取得した情報を元に WebAuthn - `navigator.credentials.get()`を実行する
4. 認証器で`authenticatorGetCredential()`が実行され、本人性の検証を行い、Assestation[^2] が生成される
5. 認証器で生成された情報がクライアント(WebAuthn Client)へ返却される
6. 認証器から取得した情報のうち、検証に必要な情報(clientDataJSON, authenticatorData, signature)をRPへ対して送信する
7. RPで認証器で生成された情報の検証を行う

[^2]: 適切な秘密鍵を所持している事を証明するための情報のこと



こちらも4, 5に関しては今回の対象外とさせてもらいます。また、通常4, 5に関してはWebAuthn + CTAPで規定されており、各対応ブラウザ(+ 認証器)が良しなに処理してくれるので、実は実装上あまり意識しなくてもよい箇所ではあります。(4, 5に関して深掘りした記事はいつか書こうと思います...いつか...)



### 1. ユーザーの認証要求

プロトコル、フォーマット等は決まっていないので自分で決める必要があります。今回は、`GET /webauthn/assertion/options/{email}`でリクエストを発行しています。

```typescript
// http://localhost:8081/webauthn/assertion/options/{email}
this.httpClient.get<AssertionServerOptions>(`${this.ASSERTION_OPTION}/${email}`, {
    headers: {
        'Content-type': 'application/json',
    },
}).toPromise()
```



### 2. `navigator.credentials.get()`実行に必要なパラメータの生成

`navigator.credentials.get()`の実行に必要なパラメータを生成します。この時、必ず必要となるパラメータはchallengeのみですが今回は以下のように実装しています。



```java
var user = entityManager.find(Users.class, email);
var challenge = new DefaultChallenge();  // ... 1
user.setChallenge(challenge.getValue()); // ... 1
var allowCredentials = entityManager.createNamedQuery("getCredentialById", Credentials.class) // ... 2
    .setParameter("credentialId", user.getCredentialId())
    .getResultStream()
    .map(credential -> new PublicKeyCredentialDescriptor(
        PublicKeyCredentialType.PUBLIC_KEY,
        Base64UrlUtil.decode(credential.getCredentialId()),
        new HashSet<>(Arrays.asList(AuthenticatorTransport.USB,
                                    AuthenticatorTransport.BLE,
                                    AuthenticatorTransport.INTERNAL,
                                    AuthenticatorTransport.NFC)
                     )
    )).collect(Collectors.toList());
return new PublicKeyCredentialRequestOptions( // ... 3
    challenge,
    TimeUnit.SECONDS.toMillis(60),
    rp.getId(),
    allowCredentials, 
    UserVerificationRequirement.PREFERRED,
    null
);
```



#### 1. challenge

認証フローも最初にリプライ攻撃防止用のchallengeを生成します。例によって、[WebAuthn4J](https://github.com/webauthn4j/webauthn4j)の`com.webauthn4j.data.client.challenge.DefaultChallenge`を使用してchallengeを生成します。また、フローの最後の検証時に認証器で署名されたchallengeとRPで生成されたchallengeが一致するかどうかの検証を行うため、どこかに保存しておきます。(今回はDatabase)



#### 2. allowCredential

ユーザーに紐づくCredential IDのリストを指定するオプションです。それぞれのプロパティは以下のような意味を持ちます。

| プロパティ | 概要                                                       |
| ---------- | ---------------------------------------------------------- |
| id         | credentialId                                               |
| type       | `public-key`で固定                                         |
| transports | 認証器との接続に使用できると思われるトランスポートのヒント |



#### 3. PublicKeyCredentialRequestOptionsを返却する

こちらもクライアントに返却する際にchallenge等のバイト列は、Base64urlエンコードする必要があるので、レスポンス専用のクラスを生成します。今回はこんな感じで。

```AssertionServerOptions.java
import com.webauthn4j.data.UserVerificationRequirement;
import com.webauthn4j.data.extension.client.AuthenticationExtensionClientInput;
import com.webauthn4j.data.extension.client.AuthenticationExtensionsClientInputs;

import java.io.Serializable;
import java.util.List;

public class AssertionServerOptions implements Serializable {
    public String challenge;
    public Long timeout;
    public String rpId;
    public List<MyPublicKeyCredentialDescriptor> allowCredentials;
    public UserVerificationRequirement userVerification;
    public AuthenticationExtensionsClientInputs<AuthenticationExtensionClientInput> extensions;

    public AssertionServerOptions() {
    }

    public AssertionServerOptions(String challenge, Long timeout, String rpId, List<MyPublicKeyCredentialDescriptor> allowCredentials, UserVerificationRequirement userVerification, AuthenticationExtensionsClientInputs<AuthenticationExtensionClientInput> extensions) {
        this.challenge = challenge;
        this.timeout = timeout;
        this.rpId = rpId;
        this.allowCredentials = allowCredentials;
        this.userVerification = userVerification;
        this.extensions = extensions;
    }
}

```

次に生成した`PublicKeyCredentialRequestOptions`からレスポンス用のオブジェクトに詰め替えてクライアントに返却します。お好きな方法でどうぞ。参考までに。

```java
@Path("assertion/options/{email}")
@Produces(MediaType.APPLICATION_JSON)
public AssertionServerOptions assertionOptions(@PathParam("email") String email) {
    var publicKeyCredentialRequestOptions = webAuthnService.requestServerOptions(email);
    return new AssertionServerOptions(
        Base64UrlUtil.encodeToString(publicKeyCredentialRequestOptions.getChallenge().getValue()),
        publicKeyCredentialRequestOptions.getTimeout(),
        publicKeyCredentialRequestOptions.getRpId(),
        publicKeyCredentialRequestOptions.getAllowCredentials()
        .stream()
        .map(publicKeyCredentialDescriptor -> new MyPublicKeyCredentialDescriptor(
            publicKeyCredentialDescriptor.getType().getValue(),
            Base64UrlUtil.encodeToString(publicKeyCredentialDescriptor.getId()),
            publicKeyCredentialDescriptor.getTransports())
            ).collect(Collectors.toList()),
        publicKeyCredentialRequestOptions.getUserVerification().getValue(),
        publicKeyCredentialRequestOptions.getExtensions()
    );
}
```



### 3. `navigator.credential.get()`を実行する

RPから取得した情報を元に`navigator.credentials.get()`の実行に必要なパラメータを組み立てて実行します。

```typescript
  public async requestCredential(email: string): Promise<Credential | null> {
    return this.fetchAssertionOptions(email).then((fetchedOptions) => {
      const credentialRequestOptions: CredentialRequestOptions = {
        publicKey: fetchedOptions,
      };
      console.log('credentialRequestOptions', credentialRequestOptions);
      return navigator.credentials.get(credentialRequestOptions);
    });
  }  
  
  private async fetchAssertionOptions(
    email: string
  ): Promise<PublicKeyCredentialRequestOptions> {
    console.log(email);
    return this.httpClient
      .get<AssertionServerOptions>(`${this.ASSERTION_OPTION}/${email}`)
      .toPromise()
      .then((requestOptions) => {
        return {
          // require
          challenge: Base64urlUtil.base64urlToArrayBuffer(
            requestOptions.challenge
          ),
          // option
          timeout: requestOptions.timeout,
          rpId: requestOptions.rpId,
          allowCredentials: requestOptions.allowCredentials?.map(
            (allowCredential) => {
              return {
                type: allowCredential.type,
                id: Base64urlUtil.base64urlToArrayBuffer(allowCredential.id),
                transports: allowCredential.transports,
              };
            }
          ),
          userVerification: requestOptions.userVerification,
          extensions: requestOptions.extensions,
        };
      });
  }
```





### 6. 認証器が生成した情報から必要な情報をRPへ送信する

認証器の`authenticatorGetCredential()`が実行されると最終的に以下のようなデータが戻ってきます。

```json
{
  "id": "bLYmTeblx08JDdRZc…",
  "rawId": ArrayBuffer(64) {},
  "response": {
    "authenticatorData": ArrayBuffer(37),
    "clientDataJSON": ArrayBuffer(222) {},
    "signature": ArrayBuffer(70) {}
  },
  "type": "public-key"
}

```

それぞれのプロパティを見ていきましょう。

| パラメータ名      | 概要                                                         |
| ----------------- | ------------------------------------------------------------ |
| id                | rawId を base64url エンコードしたもの                        |
| rawId             | Credential 毎に一意に定められている                          |
| authenticatorData | 認証器で生成された情報(rpIdHash, ユーザーの検証結果などが含まれる) |
| clientDataJSON    | challenge, oritin, type などが含まれている clientData を JSON シリアライズしたもの |
| signature         | 登録時に生成された秘密鍵を用いて作成された署名               |
| type              | `public-key`固定                                             |

このうち、`authenticatorData`, `clientDataJSON`, `signature`を Relying Party へ送信し検証が成功すれば認証処理は完了です。

```typescript
const credential = navigator.credentials.get(credentialRequestOptions);
const publicKeyCredential: PublicKeyCredential = credential as PublicKeyCredential;
const assertionResponse: AuthenticatorAssertionResponse = publicKeyCredential.response as AuthenticatorAssertionResponse;
const credentialId = publicKeyCredential.rawId;
const clientDataJSON = assertionResponse.clientDataJSON;
const authenticatorData = assertionResponse.authenticatorData;
const signature = assertionResponse.signature;
const userHandle = assertionResponse.userHandle;
this.httpClient.post<AssertionResult>(
  `${this.ASSERTION_REQUEST}`, // http://localhost:8081/webauthn/assertion/result
  {
    credentialId: Base64urlUtil.arrayBufferToBase64url(credentialId),
    clientDataJSON: Base64urlUtil.arrayBufferToBase64url(clientDataJSON),
    authenticatorData: Base64urlUtil.arrayBufferToBase64url(
      authenticatorData
    ),
    signature: Base64urlUtil.arrayBufferToBase64url(signature),
    userHandle: Base64urlUtil.arrayBufferToBase64url(userHandle),
  }).toPromise();
```



### 7. RPで認証器から取得した情報の検証を行う

認証器で生成された情報の検証は

- Relying PartyのIDが期待されたものか
- 認証器で署名されたchallengeと認証フローの最初にRPで生成したchallengeが一致するか
- 登録処理の際に保存した公開鍵を使用して、署名が検証できるか

ということを行いますが、`com.webauthn4j.WebAuthnManager#validate(com.webauthn4j.data.AuthenticationRequest, com.webauthn4j.data.AuthenticationParameters)`というメソッドでこれらの検証を実施してくれるのでそのために必要なパラメータを組み立てます。一連の流れは以下の通り。

```java
var origin = Origin.create("http://localhost");
var serverProperty = entityManager.createNamedQuery("getUserByCredentialId", Users.class)
    .setParameter("credentialId", Base64UrlUtil.encodeToString(credentialId))
    .getResultStream()
    .map((users -> new ServerProperty(origin, rp.getId(), new DefaultChallenge(users.getChallenge()), null)))
    .collect(Collectors.toList())
    .get(0);
var authenticators = entityManager.createNamedQuery("getCredentialById", Credentials.class)
    .setParameter("credentialId", Base64UrlUtil.encodeToString(credentialId))
    .getResultList();
if (authenticators.isEmpty()) {
    // do something
}
var authenticationRequest = new AuthenticationRequest(credentialId, userHandle, authenticatorData, clientDataJSON, signature);
var authenticationParameter = new AuthenticationParameters(serverProperty, getAuthenticator(credentialId), false);
try {
    WebAuthnManager.createNonStrictWebAuthnManager().parse(authenticationRequest);
} catch (DataConversionException e) {
    // do something
}
try {
    // 認証器から取得した情報の検証
    WebAuthnManager.createNonStrictWebAuthnManager().validate(authenticationRequest, authenticationParameter);
} catch (ValidationException e) {
    // do something
}
```



# 参考

- [MDN - Web Authentication API](https://developer.mozilla.org/ja/docs/Web/API/Web_Authentication_API)
- [Web Authentication: An API for accessing Public Key Credentials Level 1](https://www.w3.org/TR/webauthn-1/)

- [WebAuthn4J](https://github.com/webauthn4j/webauthn4j)
