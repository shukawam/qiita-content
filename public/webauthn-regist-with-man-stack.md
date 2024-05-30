---
title: MANスタックでWebAuthn - ユーザ登録編
tags:
  - Node.js
  - Angular
  - NestJS
  - WebAuthn
  - FIDO2
private: false
updated_at: '2021-09-30T22:05:59+09:00'
id: 80b89437add0628af2a3
organization_url_name: null
slide: false
ignorePublish: false
---
# 始めに

この記事は、WebAuthn を使用したユーザの登録フローに関する学習メモです。作成したソースコード一式は[こちら](https://github.com/shukawam/sandbox/tree/master/fido2-man-stuck-sample)に格納しておきます。

# FIDO2 について

## TL; DR

- FIDO Alliance という非営利団体が推進する認証技術、規格群の一つ
- 従来の生体認証などで行われていた専用の機器などを用いずに、Web からパスワードレス認証することを目的とした認証規格
- FIDO2 = WebAuthn + CTAP2
    - FIDO U2F に加え、Web から FIDO UAF を使う仕組みとも解釈できる

## W3C WebAuthn

FIDO 認証のサポートを可能にするためにブラウザおよびプラットフォームに組み込まれている標準 Web API のこと。登録と認証の機能を持つ。

- `navigator.credentials.create()`: publicKey オプションと併用すると、新規に認証情報を作成します。
- `navigator.credentials.get()`: publicKey オプションと併用すると、既存の認証情報を取得します。



## CTAP（Client to Authentication Protocol）

名前の通り、クライアント(Client)と認証器(Authenticator)間の通信プロトコルです。
Relying Party を実装する上では、[CTAPの仕様](https://fidoalliance.org/specs/fido-v2.0-ps-20190130/fido-client-to-authenticator-protocol-v2.0-ps-20190130.html)に関する理解は不要ですが、覗いてみると結構楽しいです。

## 仕組みの概要

FIDOのプロトコルでは、標準的な公開鍵暗号方式を用いて、認証を実現しています。
以下、基本的な処理シーケンス。

![authentication-sequence.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/a0890b97-92d1-7e18-0361-14bffb9256ed.png)

- クライアントとサーバ間でパスワード等の認証情報をやり取りしないため、従来のID/Password方式の認証方式よりも安全だと言われている。
- クライアント側に認証に必要な秘密鍵を保持することで、ユーザがパスワードを記憶する必要がない。

## サービスの認証に FIDO2 を導入するためには

1. Relying Party を自作する
2. Relying Party を実装している IDaaS を使用する

と、大きく2通りの方法がありますが今回は学習目的のため、自分で実装します。

# Relying Party を自作してみる

## 実装言語・FW

- 認証サーバ: Nest.js v6.13.3 
- クライアント: Angular v9.0.4 
- Database: MongoDB

の MAN スタックで実装してみました。お好きな言語、FWで作ってみてください。

## 登録の処理シーケンス

![WebAuthn_Registration_r4.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/d5a8be06-febb-6568-9f29-127f90b3876b.png)

※[MDM](https://developer.mozilla.org/ja/docs/Web/API/Web_Authentication_API)より引用

## 処理概要

0. 認証サーバに対して、challenge の生成をリクエストする。

1. 認証サーバで生成した challenge, ユーザ情報、サーバの情報をクライアントにレスポンスする。

2. 取得したデータを元にパラメータを組み立て、`navigator.credentials.create()`を呼び出す。
3. 非対称鍵ペア（公開鍵と秘密鍵）と Attestation を生成する。（Attestation: 公開鍵がユーザが所持する認証器から生成されたものであることを保証するための仕組み）
4. 生成したデータをクライアントに返却する。
5. 認証器が生成した情報を Relying Party に送信する。
6. 認証器が生成した情報の検証を行う。

WebAuthn で規定されていない箇所（上図の0, 1, 5, 6）に関しては自分で仕様を考えて実装する必要があります。

従って、今回作成するのは以下の赤枠部分となります。（WebAuthn - Relying Party と JavaScript Application）

![WebAuthn_Registration_r4_change.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/17c4c269-269b-ffd6-ee88-2ca6212860c1.png)

## 実装のポイント

全部を載せると、とんでもない量になってしまうのでかいつまんでポイントを説明します。実装の細かい点は、[リポジトリ](https://github.com/shukawam/sandbox/tree/master/fido2-man-stuck-sample)を参照してください。

### 0. challengeの生成リクエスト ~ 1. challenge、ユーザ情報、サーバ情報のレスポンス

WebAuthn によって仕様が定義されているわけではないため、自分で設計をする必要があります。

今回は、クライアントから以下のようなリクエストを発行してみました。

```http
POST http://localhost:3000/webauthn/register HTTP/1.1
Content-Type: application/json

{
  "email": "test-user-001@example.com"
}
```

それを受ける認証サーバは以下のように実装しています。

```webauthn.controller.ts
@Controller('webauthn')
export class WebauthnController {

  constructor(private readonly webauthnSercice: WebauthnService) { }

  /**
   * challenge生成のエンドポイントです。
   * @param createUserDto リクエストボディー
   */
  @Post('/register')
  async register(@Body() createUserDto: CreateUserDto): Promise<ResponseData> {
    const userCreationOptions = await this.webauthnSercice.createUserCreationOptions(createUserDto);
    if (!userCreationOptions) {
      throw new HttpException({
        status: HttpStatus.INTERNAL_SERVER_ERROR,
        error: 'database error.',
      }, HttpStatus.INTERNAL_SERVER_ERROR);
    }
    const responseData = new ResponseData();
    responseData.status = HttpStatus.CREATED;
    responseData.data = userCreationOptions;
    return responseData;
  }

  // ... 省略
}

```

```webauthn.service.ts
@Injectable()
export class WebauthnService {

  private readonly ORIGIN = 'http://localhost:4200';

  constructor(@InjectModel('User') private userModel: Model<User>) { }

  /**
   * 認証器が鍵の生成に必要なパラメータを生成します。
   * @param createUserDto リクエストボディー
   */
  async createUserCreationOptions(createUserDto: CreateUserDto): Promise<UserCreationOptions> {
    // 少なくとも16バイト以上のランダムに生成されたバッファーを生成する
    const challenge = Buffer.from(Uint8Array.from(uuid(), c => c.charCodeAt(0)));
    const userId = Buffer.from(Uint8Array.from(uuid(), c => c.charCodeAt(0)));
    const userCreationOptions: UserCreationOptions = {
      email: createUserDto.email,
      challenge: base64url.encode(challenge),
      rp: {
        name: 'webauthn-server-nestjs-sample',
      },
      user: {
        id: base64url.encode(userId),
        name: createUserDto.email,
        displayName: createUserDto.email,
      },
      attestation: 'direct',
    };
    // DBに保存する
    const saveResult = await this.saveUser(userCreationOptions);
    if (!saveResult) {
      return null;
    }
    return userCreationOptions;
  }

  /**
   * ユーザをDBに保存します。
   * @param userCreationOptions ユーザの認証情報
   */
  private async saveUser(userCreationOptions: UserCreationOptions): Promise<User> {
    // ユーザが保存済みがどうか確認する
    const user = await this.userModel.findOne({ email: userCreationOptions.email }).exec();
    if (user) {
      throw new HttpException({
        status: HttpStatus.CONFLICT,
        error: 'user already exists.',
      }, HttpStatus.CONFLICT);
    }
    const newUser = new this.userModel(userCreationOptions);
    return newUser.save();
  }

}

```

ポイントは２つあります。

-  challenge は、認証サーバで生成する。また、生成する challenge は、少なくとも１６バイト以上でランダムに生成されたバッファであること。

これを満たすために、今回はuuid(v4)を元にバッファを生成しています。

```tsx
const challenge = Buffer.from(Uint8Array.from(uuid(), c => c.charCodeAt(0)));
```

- 特に定められていないが、レスポンスはWebAuthn APIで扱いやすい形式で返却するほうが望ましい。

これを踏まえて、今回は以下のようなレスポンスをクライアントに返却しています。

```http
HTTP/1.1 201 Created
X-Powered-By: Express
Content-Type: application/json; charset=utf-8
Content-Length: 333
ETag: W/"14d-LWc+sLb+7AIGIewNEbfdcmI1pHw"
Date: Mon, 01 Jun 2020 14:28:49 GMT
Connection: close

{
  "status": 201,
  "data": {
    "email": "test-user-001@example.com",
    "challenge": "MTJjMGUzMmEtMzM3My00ODAzLThiMTMtZGU3YmFhMzdhZWY5",
    "rp": {
      "name": "webauthn-server-nestjs-sample"
    },
    "user": {
      "id": "MjA4YTI3NWQtYmFhYi00ZDQyLTliODEtMWNmMzQ1NjMxYTY1",
      "name": "test-user-001@example.com",
      "displayName": "test-user-001@example.com"
    },
    "attestation": "direct"
  }
}
```

| パラメータ  | 概要説明                                                     |
| :---------- | :----------------------------------------------------------- |
| challenge   | 署名の正当性を検証するためのランダムな文字列。サーバで生成したランダムバッファをbase64urlエンコードしたもの。 |
| rp          | 認証サーバの情報                                             |
| user        | ユーザの登録情報                                             |
| attestation | 認証からCredentialをどのように受け取るかを記したもの。<br />`direct`の他にも`none`や`indirect`といったパラメータが存在する。<br />詳細は、[Attestation Conveyance Preference Enumeration](https://www.w3.org/TR/webauthn/#enumdef-attestationconveyancepreference)を参照してください。 |

### 2. `navigator.credentials.create()`呼び出し ~ 5. サーバに送信

認証サーバから取得したデータを元に`navigator.credentials.create()`呼び出しに必要なパラメータを作成します。

```typescript
// challengeの生成要求
const registerResponse = await this.createChallenge(email);
// navigator.credentials.create()呼び出しのために必要なパラメータの組み立て
const publicKeyCredentialCreationOptions: PublicKeyCredentialCreationOptions = {
    challenge: Buffer.from(base64url.decode(registerResponse.data.challenge)),
    rp: registerResponse.data.rp,
    user: {
        id: Buffer.from(base64url.decode(registerResponse.data.user.id)),
        name: registerResponse.data.user.name,
        displayName: registerResponse.data.user.displayName,
    },
    attestation: registerResponse.data.attestation,
    pubKeyCredParams: [{
        type: 'public-key' as 'public-key',
        alg: -7,
    }],
    authenticatorSelection: {
        authenticatorAttachment: 'cross-platform',
        requireResidentKey: false,
        userVerification: 'discouraged'
    }
};
// ... 省略
```

| パラメータ             | 概要説明                                                     |
| ---------------------- | ------------------------------------------------------------ |
| pubKetCredParams       | 認証器の鍵作成に用いるアルゴリズムを指定する。今回は、-7 (ECDSA-SHA256)を指定しています。 |
| authenticatorSelection | 認証器の種類を限定できる。今回は、Yubikeyのようなクロスプラットフォームの認証器を使用したかったため、`cross-platform`を指定しています。 |

`navigator.credentials.create()`を呼び出し、そのレスポンスを認証サーバに送信する。という一連の流れが以下になります。

```sign-up.component.ts
@Injectable({
  providedIn: 'root'
})
export class SignUpService {

  constructor(private readonly httpClient: HttpClient) { }

  /**
   * ユーザの登録処理を実行します。
   * @param email メールアドレス
   */
  async signUp(email: string): Promise<boolean> {
    // challengeの生成要求
    const registerResponse = await this.createChallenge(email);
    // `navigator.credentials.create()呼び出しのために必要なパラメータの組み立て
    const publicKeyCredentialCreationOptions: PublicKeyCredentialCreationOptions = {
      challenge: Buffer.from(base64url.decode(registerResponse.data.challenge)),
      rp: registerResponse.data.rp,
      user: {
        id: Buffer.from(base64url.decode(registerResponse.data.user.id)),
        name: registerResponse.data.user.name,
        displayName: registerResponse.data.user.displayName,
      },
      attestation: registerResponse.data.attestation,
      pubKeyCredParams: [{
        type: 'public-key' as 'public-key',
        alg: -7,
      }],
      authenticatorSelection: {
        authenticatorAttachment: 'cross-platform',
        requireResidentKey: false,
        userVerification: 'discouraged'
      }
    };
    // 明示的にPublicKeyCredentialにキャストする
    const attestationObject = await this.createAttestationObject(publicKeyCredentialCreationOptions) as PublicKeyCredential;
    console.log(attestationObject);
    // 公開鍵をサーバに送信する
    return this.registerPublicKey(attestationObject);
  }

  /**
   * WebAuthn認証サーバに対して、チャレンジの生成要求を行います。
   * @param email メールアドレス
   */
  private async createChallenge(email: string): Promise<User> {
    const registerResponse = await this.httpClient.post<User>(Uri.USER_REGISTER, { email }, {
      headers: {
        'Content-Type': 'application/json'
      },
      observe: 'response',
    }).toPromise();
    console.log(registerResponse.body);
    return registerResponse.body;
  }

  /**
   * 認証器に対して公開鍵の生成要求を行います。
   * @param publicKeyCreationOptions 認証情報生成オプション
   */
  private async createAttestationObject(publicKeyCreationOptions: PublicKeyCredentialCreationOptions): Promise<Credential> {
    return navigator.credentials.create({
      publicKey: publicKeyCreationOptions
    });
  }

  /**
   * 認証情報をBase64Urlエンコードして認証サーバにPOSTします。
   * @param credential 認証器で生成した認証情報
   */
  private async registerPublicKey(publicKeyCredential: PublicKeyCredential): Promise<boolean> {
    const attestationResponse = await this.httpClient.post(Uri.ATTESTATION_RESPONSE,
      {
        rawId: base64url.encode(Buffer.from(publicKeyCredential.rawId)),
        response: {
          attestationObject: base64url.encode(Buffer.from(publicKeyCredential.response.attestationObject)),
          clientDataJSON: base64url.encode(Buffer.from(publicKeyCredential.response.clientDataJSON)),
        },
        id: publicKeyCredential.id,
        type: publicKeyCredential.type
      }, {
      headers: {
        'Content-Type': 'application/json'
      },
      observe: 'response'
    }).toPromise();
    return attestationResponse.body ? true : false;
  }
}

```

### 6. 認証情報のチェック

クライアントから以下のようなリクエストが送信されてきます。

```http
POST http://localhost:3000/webauthn/response HTTP/1.1
Content-Type: application/json

{
   "rawId":"MLzRnn5P7mRPTK5sATEmKiJfhvV1TJgGXHCbWu3mKrcSZW-oZQ4LwZ3kqeN6KRfESWDbJfv8EXdXHr53XhOQiAvV1Gti4XR9gJaQY45HQK_xw98VxP7e9EnOLjdi6_5a3nLs4lAkQjJ1TqY4IJBnFNSbue7nUAotIQ6kD3ubYR5S",
   "response":{
      "attestationObject":"o2NmbXRoZmlkby11MmZnYXR0U3RtdKJjc2lnWEcwRQIgO3T6_LkyjbSDnIyWX29oe7dUflpm6nt2BB9U1sdVcTwCIQDacpQ3-TAMhaTsFPM039VvjHqSQDUFzC_YaYHkk88v72N4NWOBWQKpMIICpTCCAkqgAwIBAgIJANhaddxx4y8sMAoGCCqGSM49BAMCMIGlMQswCQYDVQQGEwJDTjESMBAGA1UECAwJR3Vhbmdkb25nMREwDwYDVQQHDAhTaGVuemhlbjEzMDEGA1UECgwqU2hlbnpoZW4gRXhjZWxzZWN1IERhdGEgVGVjaG5vbG9neSBDby4gTHRkMR4wHAYDVQQLDBVFeGNlbHNlY3UgRmlkbyBTZXJ2ZXIxGjAYBgNVBAMMEUV4Y2Vsc2VjdSBGaWRvIENBMB4XDTE4MDExOTAzNDY1OVoXDTI4MDExNzAzNDY1OVowgawxCzAJBgNVBAYTAkNOMRIwEAYDVQQIDAlHdWFuZ2RvbmcxETAPBgNVBAcMCFNoZW56aGVuMTMwMQYDVQQKDCpTaGVuemhlbiBFeGNlbHNlY3UgRGF0YSBUZWNobm9sb2d5IENvLiBMdGQxHjAcBgNVBAsMFUV4Y2Vsc2VjdSBGaWRvIFNlcnZlcjEhMB8GA1UEAwwYRXhjZWxzZWN1IEZpZG8gVTJGIDAwMDAyMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEtwOC4SZp2EpDMVxiZS-P_2wp_ZBNMEFKTruWGdg38qM4r_jT5r_a1vxW0UN89LFY1m1BpXuUAeeCn36DriitcaNaMFgwCQYDVR0TBAIwADALBgNVHQ8EBAMCB4AwHQYDVR0OBBYEFERWGpXZomZNMqJn2_6GzguxnlkmMB8GA1UdIwQYMBaAFKyJLw-sy4g7nHYTZwKpZqyJzZ-bMAoGCCqGSM49BAMCA0kAMEYCIQCpPai4VwA59-PiHq8SYjS9qcffQD-3oFnfR9njRpY5UwIhAMlMszhSeaf0xaAPC48ZYSB_ZeZ8vgnkQOFjfctD_EFmaGF1dGhEYXRhWQEFSZYN5YgOjGh0NBcPZHZgW4_krrmihjLHmVzzuoMdl2NBAAAAAAAAAAAAAAAAAAAAAAAAAAAAgTC80Z5-T-5kT0yubAExJioiX4b1dUyYBlxwm1rt5iq3EmVvqGUOC8Gd5KnjeikXxElg2yX7_BF3Vx6-d14TkIgL1dRrYuF0fYCWkGOOR0Cv8cPfFcT-3vRJzi43Yuv-Wt5y7OJQJEIydU6mOCCQZxTUm7nu51AKLSEOpA97m2EeUqUBAgMmIAEhWCBwt4oPucNcbc8PIR7gFdM9tWAr0NCKc9HjzPvB4h0wvSJYIK09jRBM_VY8ms4y5pnsfURZjwTcvmu6noWK7GXpCNxy",
      "clientDataJSON":"eyJ0eXBlIjoid2ViYXV0aG4uY3JlYXRlIiwiY2hhbGxlbmdlIjoiWlRjNE4yUTNZbUV0TUROaU15MDBaVGxtTFdFek1EWXROamhtTTJRek1UQXdPV1JpIiwib3JpZ2luIjoiaHR0cDovL2xvY2FsaG9zdDo0MjAwIiwiY3Jvc3NPcmlnaW4iOmZhbHNlfQ"
   },
   "id":"MLzRnn5P7mRPTK5sATEmKiJfhvV1TJgGXHCbWu3mKrcSZW-oZQ4LwZ3kqeN6KRfESWDbJfv8EXdXHr53XhOQiAvV1Gti4XR9gJaQY45HQK_xw98VxP7e9EnOLjdi6_5a3nLs4lAkQjJ1TqY4IJBnFNSbue7nUAotIQ6kD3ubYR5S",
   "type":"public-key"
}
```

| パラメータ | 概要説明                                                     |
| ---------- | ------------------------------------------------------------ |
| rawId      | 公開鍵のID。                                                 |
| response   | 認証器が生成した情報。attestationObject, clientDataJSONというパラメータを持ち、認証器が生成した情報を検証する際に使用する。 |
| id         | rawIdをbase64urlエンコードしたもの。                         |
| type       | 'public-key'固定                                             |


それを受ける認証サーバは以下のように実装しています。

```webauthn.controller.ts
@Controller('webauthn')
export class WebauthnController {

  constructor(private readonly webauthnSercice: WebauthnService) { }

  // ... 省略

  /**
   * 認証器で生成した認証情報を受け取るエンドポイントです。
   * @param createCredentialDto リクエストボディー
   */
  @Post('/response')
  async response(@Body() createCredentialDto: CreateCredentialDto): Promise<ResponseData> {
    const verifyResult = await this.webauthnSercice.isValidCredential(createCredentialDto);
    const responseData = new ResponseData();
    verifyResult ? responseData.status = HttpStatus.OK : responseData.status = HttpStatus.INTERNAL_SERVER_ERROR;
    return responseData;
  }

  // 省略
}
```

```webauthn.service.ts
@Injectable()
export class WebauthnService {

  private readonly ORIGIN = 'http://localhost:4200';

  constructor(@InjectModel('User') private userModel: Model<User>) { }
  
  // ... 省略

  /**
   * 認証器が生成した認証情報の検証を行います。
   * @param createCredentialDto 認証器が生成した認証情報
   */
  async isValidCredential(createCredentialDto: CreateCredentialDto): Promise<boolean> {
    // clientDataJSONをデコードし、JSON形式にパースする
    const clientData: DecodedClientDataJson = JSON.parse(base64url.decode(createCredentialDto.response.clientDataJSON));
    Logger.debug(clientData, 'WebAuthnService', true);
    // originの検証を行う
    if (clientData.origin !== this.ORIGIN) {
      throw new HttpException('Origin is not correct.', HttpStatus.BAD_REQUEST);
    }
    // challengeの検証を行う
    const count = await this.userModel.findOne({ challenge: Buffer.from(clientData.challenge) }).count();
    Logger.debug(count, 'webauthnService#isvalidCredential', true);
    if (count === 0) {
      throw new HttpException('Challenge is not collect.', HttpStatus.BAD_REQUEST);
    }
    // attestationObjectの検証を行う
    const validateResult = await this.verifyAuthenticatorAttestationResponse(createCredentialDto);
    // 公開鍵をDBに登録する
    this.userModel.findOneAndUpdate({ challenge: Buffer.from(clientData.challenge) }, { $set: { id: createCredentialDto.id } }, error => {
      if (error) {
        Logger.error(error);
        throw new Error('Update failed.');
      }
    });
    return validateResult.verified;
  }

  /**
   * AttestationObjectの検証を行います。
   * @param createCredentialDto 認証器が生成した認証データ
   */
  private async verifyAuthenticatorAttestationResponse(createCredentialDto: CreateCredentialDto): Promise<VerifiedAuthenticatorAttestationResponse> {
    // 認証器でbase64urlエンコードされているので、認証サーバでデコードする
    const attestationBuffer = base64url.toBuffer(createCredentialDto.response.attestationObject);
    // attestationObjectをCBORデコードする
    const ctapMakeCredentialResponse: CborParseAttestationObject = Decoder.decodeAllSync(attestationBuffer)[0];
    Logger.debug(ctapMakeCredentialResponse, 'WebAuthnService', true);
    const response: VerifiedAuthenticatorAttestationResponse = {
      verified: false,
    };
    if (ctapMakeCredentialResponse.fmt === 'fido-u2f') {
      const authDataStruct = this.parseMakeCredAuthData(ctapMakeCredentialResponse.authData);
      if (!authDataStruct.flags) {
        throw new Error('User was NOT presented durring authentication!');
      }
      const clientDataHash = crypto.createHash('SHA256').update(base64url.toBuffer(createCredentialDto.response.clientDataJSON)).digest();
      const reservedByte = Buffer.from([0x00]);
      const publicKey = this.convertToRawPkcsKey(authDataStruct.cosePublicKey);
      const signatureBase = Buffer.concat([reservedByte, authDataStruct.rpIdHash, clientDataHash, authDataStruct.credID, publicKey]);
      const pemCertificate = this.convertPemTextFormat(ctapMakeCredentialResponse.attStmt.x5c[0]);
      const signature = ctapMakeCredentialResponse.attStmt.sig;
      response.verified = this.verifySignature(signature, signatureBase, pemCertificate);
      const validateResult = this.verifySignature(signature, signatureBase, pemCertificate);
      // Attestation Signatureの有効性を検証する
      return validateResult ? {
        verified: validateResult,
        authInfo: {
          fmt: 'fido-u2f',
          publicKey: base64url.encode(publicKey),
          counter: authDataStruct.counter,
          credId: base64url.encode(authDataStruct.credID),
        },
      }
        : response;
    }
  }

  /**
   * AuthDataをCBORパースします。
   * @param authData 認証器の信頼性、セキュリティ等のバイナリデータ
   */
  private parseMakeCredAuthData(authData: Buffer): CborParseAuthData {
    const rpIdHash = authData.slice(0, 32);
    authData = authData.slice(32);
    const flagsBuf = authData.slice(0, 1);
    authData = authData.slice(1);
    const flags = flagsBuf[0];
    const counterBuf = authData.slice(0, 4);
    authData = authData.slice(4);
    const counter = counterBuf.readUInt32BE(0);
    const aaguid = authData.slice(0, 16);
    authData = authData.slice(16);
    const credIDLenBuf = authData.slice(0, 2);
    authData = authData.slice(2);
    const credIDLen = credIDLenBuf.readUInt16BE(0);
    const credID = authData.slice(0, credIDLen);
    authData = authData.slice(credIDLen);
    const cosePublicKey = authData;
    return {
      rpIdHash,
      flagsBuf,
      flags,
      counter,
      counterBuf,
      aaguid,
      credID,
      cosePublicKey,
    } as CborParseAuthData;
  }

  /**
   * COSEエンコードされた公開鍵をPKCS ECDHA Keyに変換します。
   * @param cosePublicKey COSEエンコードされた公開鍵
   */
  private convertToRawPkcsKey(cosePublicKey: Buffer): Buffer {
    /*
     +------+-------+-------+---------+----------------------------------+
     | name | key   | label | type    | description                      |
     |      | type  |       |         |                                  |
     +------+-------+-------+---------+----------------------------------+
     | crv  | 2     | -1    | int /   | EC Curve identifier - Taken from |
     |      |       |       | tstr    | the COSE Curves registry         |
     |      |       |       |         |                                  |
     | x    | 2     | -2    | bstr    | X Coordinate                     |
     |      |       |       |         |                                  |
     | y    | 2     | -3    | bstr /  | Y Coordinate                     |
     |      |       |       | bool    |                                  |
     |      |       |       |         |                                  |
     | d    | 2     | -4    | bstr    | Private key                      |
     +------+-------+-------+---------+----------------------------------+
  */
    const coseStruct = Decoder.decodeAllSync(cosePublicKey)[0];
    const tag = Buffer.from([0x00]);
    const x = coseStruct.get(-2);
    const y = coseStruct.get(-3);
    return Buffer.concat([tag, x, y]);
  }

  /**
   * バイナリ形式の公開鍵をOpenSSL PEM text形式に変換します。
   * @param publicKeyBuffer バイナリの公開鍵
   */
  private convertPemTextFormat(publicKeyBuffer: Buffer): string {
    if (!Buffer.isBuffer(publicKeyBuffer)) {
      throw new Error('publicKeyBuffer must be Buffer.');
    }
    let type;
    if (publicKeyBuffer.length === 65 && publicKeyBuffer[0] === 0x04) {
      publicKeyBuffer = Buffer.concat([
        Buffer.from('3059301306072a8648ce3d020106082a8648ce3d030107034200', 'hex'),
        publicKeyBuffer,
      ]);
      type = 'PUBLIC KEY';
    } else {
      type = 'CERTIFICATE';
    }
    const b64cert = publicKeyBuffer.toString('base64');
    let pemKey = '';
    for (let i = 0; i < Math.ceil(b64cert.length / 64); i++) {
      const start = 64 * i;
      pemKey += b64cert.substr(start, 64) + '\n';
    }
    pemKey = `-----BEGIN ${type}-----\n` + pemKey + `-----END ${type}-----\n`;
    return pemKey;
  }

  /**
   * 署名の妥当性を検証します。
   * @param signature 署名
   * @param data データ
   * @param publicKey 公開鍵
   */
  private verifySignature(signature: Buffer, data: Buffer, publicKey: string): boolean {
    return crypto.createVerify('SHA256')
      .update(data)
      .verify(publicKey, signature);
  }

}

```



いくつかポイントを絞って説明します。認証サーバでは、認証器が生成した情報を以下のように検証します。

1. リクエストで受け取った challenge がサーバで生成された challenge と一致するか？
2. リクエストで受け取った origin が期待する origin と一致するか？
3. attestationObject が妥当かどうか？



#### challenge, originの検証

```typescript
// clientDataJSONをデコードし、JSON形式にパースする
const clientData: DecodedClientDataJson = JSON.parse(base64url.decode(createCredentialDto.response.clientDataJSON));
```

リクエストボディに含まれる`clientDataJSON`を base64Url デコードし、JSON にパースすると以下のような JSON を取得できます。

```json
{
  "challenge": "upYb6sib9exL7fvSfQhIEazOkBh8_YJXVPzSx0T16B0",
  "origin": "http://localhost:4200",
  "type": "webauthn.create"
}
```

従って、origin, challenge は以下のように実施しています。

```typescript
// originの検証を行う
if (clientData.origin !== this.ORIGIN) {
  // do something
}
// challengeの検証を行う
const count = await this.userModel.findOne({ challenge: Buffer.from(clientData.challenge) }).count();
Logger.debug(count, 'webauthnService#isvalidCredential', true);
if (count === 0) {
  // do something
}
```

- origin：予め期待しているoriginと一致するかどうか検証
- challenge：検索条件として、リクエストに含まれるchallengeを指定し、検索結果の数で検証

※challengeがぶつかることは想定していないです。一応、uuidから生成しているので、、

#### AttestationObjectの検証

AttestationObjectは、base64urlエンコードされているCBORとなっています。実際の構成は以下の通りとなっています。

![attestation-object.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/86cc4927-9dc5-4049-e07b-8d329beb6096.png)


※[W3C](https://www.w3.org/TR/webauthn/#fig-attStructs)より引用

検証では、AttestationObjectをパースして得られるパラメータを使用して、Attestation Signatureの有効性を検証します。

実際の検証ロジックは、[fido-seminar-webauthn-tutorial](https://slides.com/fidoalliance/jan-2018-fido-seminar-webauthn-tutorial#/28/0/0)を参考にしました。

```webauthn.service.ts
  /**
   * AttestationObjectの検証を行います。
   * @param createCredentialDto 認証器が生成した認証データ
   */
  private async verifyAuthenticatorAttestationResponse(createCredentialDto: CreateCredentialDto): Promise<VerifiedAuthenticatorAttestationResponse> {
    // 認証器でbase64urlエンコードされているので、認証サーバでデコードする
    const attestationBuffer = base64url.toBuffer(createCredentialDto.response.attestationObject);
    // attestationObjectをCBORデコードする
    const ctapMakeCredentialResponse: CborParseAttestationObject = Decoder.decodeAllSync(attestationBuffer)[0];
    Logger.debug(ctapMakeCredentialResponse, 'WebAuthnService', true);
    const response: VerifiedAuthenticatorAttestationResponse = {
      verified: false,
    };
    if (ctapMakeCredentialResponse.fmt === 'fido-u2f') {
      const authDataStruct = this.parseMakeCredAuthData(ctapMakeCredentialResponse.authData);
      if (!authDataStruct.flags) {
        throw new Error('User was NOT presented durring authentication!');
      }
      const clientDataHash = crypto.createHash('SHA256').update(base64url.toBuffer(createCredentialDto.response.clientDataJSON)).digest();
      const reservedByte = Buffer.from([0x00]);
      const publicKey = this.convertToRawPkcsKey(authDataStruct.cosePublicKey);
      const signatureBase = Buffer.concat([reservedByte, authDataStruct.rpIdHash, clientDataHash, authDataStruct.credID, publicKey]);
      const pemCertificate = this.convertPemTextFormat(ctapMakeCredentialResponse.attStmt.x5c[0]);
      const signature = ctapMakeCredentialResponse.attStmt.sig;
      response.verified = this.verifySignature(signature, signatureBase, pemCertificate);
      const validateResult = this.verifySignature(signature, signatureBase, pemCertificate);
      // Attestation Signatureの有効性を検証する
      return validateResult ? {
        verified: validateResult,
        authInfo: {
          fmt: 'fido-u2f',
          publicKey: base64url.encode(publicKey),
          counter: authDataStruct.counter,
          credId: base64url.encode(authDataStruct.credID),
        },
      }
        : response;
    }
  }
```



# 完成イメージ

![registration.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/7da06d2d-25aa-9bda-1458-db07db5a4e42.gif)




# 終わりに

自分で実装してみることで、「完全に理解した！」から「なんも分からん」くらいにはなれたと思います。

次は、認証のフローについて書きます。

2020/06/06 追記
書きました。⇒ [FIDO2（WebAuthn）に入門してみた - ユーザ認証編](https://qiita.com/s001_kawamura/items/07b6059aa5da67fef759)



# 参考

- [https://fidoalliance.org/%E4%BB%95%E6%A7%98%E6%A6%82%E8%A6%81/?lang=ja](https://fidoalliance.org/仕様概要/?lang=ja)
- [https://developer.mozilla.org/ja/docs/Web/API/Web_Authentication_API](https://developer.mozilla.org/ja/docs/Web/API/Web_Authentication_API)
- [https://tech.mercari.com/entry/2019/06/04/120000](https://tech.mercari.com/entry/2019/06/04/120000)
- [https://slides.com/fidoalliance/jan-2018-fido-seminar-webauthn-tutorial#/](
