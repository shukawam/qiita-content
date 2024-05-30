---
title: Micronaut を使って ATP と連携する Function を簡単に実装する
tags:
  - oracle
  - oci
  - Micronaut
  - autonomous_database
private: false
updated_at: '2022-12-05T09:25:49+09:00'
id: abd83a456d30ce5c689a
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

この記事は、 [JPOUG Advent Calendar 2022](https://adventar.org/calendars/7680) 5 日目の記事です。4 日目は [wmo6hash](https://adventar.org/users/4289) さんの[「働き方の新しいスタイル」への仕事道具最適化'22](https://wmo6hash.hatenablog.jp/entry/2022/12/04/000000) でした！

私の記事では、以前書いた [Micronaut on Oracle Functions ことはじめ](https://qiita.com/shukawam/items/27d448c8c9b269f22d3c) の続編ということで、ATP(Autonomous Transaction Processing)と連携する Function を実装してみたいと思います。

## 全体像

以下のような構成を作成します。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/b871c55c-0829-e039-3b4c-ba306636030e.png)

## 手順

作成済みのサンプルコードは[こちら](https://github.com/shukawam/country/tree/main/micronaut-country)に格納してあります。詳細な箇所は格納済みのサンプルコードをご参照ください。

### 前提

- OCI のアカウントを有していること
- ATP が作成済みであること
- OCI Vault が使えること
- 適切なポリシー設定がされている OCI の VM
  - Config ファイルベースで認証を行う場合は、[Micronaut - ConfigFileAuthenticationDetailsProvider](https://micronaut-projects.github.io/micronaut-oracle-cloud/latest/guide/#config-auth) をご参照ください。

### ひな形の作成

例によって、[https://micronaut.io/launch/](https://micronaut.io/launch/) を使いますが、手元に Micronaut CLI がある方はそっちを使ってもいいです。

```bash
curl \
    --location \
    --request GET 'https://launch.micronaut.io/create/default/me.shukawam.country?lang=JAVA&build=GRADLE&test=JUNIT&javaVersion=JDK_11&features=oracle-cloud-atp&features=oracle-cloud-vault&features=oracle-function&features=data-jpa' \
    --output country.zip
```

含めている features を簡単に補足しておきます。

- oracle-cloud-atp
  - ATP と接続するための Wallet のダウンロードや展開等を実施してくれる機能です
- oracle-cloud-vault
  - 機微情報（データベースの接続に用いるユーザー名、パスワード、等）を OCI Vault から参照するための機能です
- oracle-function
  - Oracle Functions として Micronaut アプリケーションを実行するための機能です
- data-jpa
  - データベースアクセスに用います
  - [Spring Data](https://spring.io/projects/spring-data) を使ったことある方でしたらかなり取っつきやすいはず

解凍します。

```bash
unzip country.zip
```

含まれているファイル群を確認します。（※説明のために一部コメントを追記しています）

```bash
country
├── README.md
├── build.gradle
├── func.yml
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradle.properties
├── gradlew
├── gradlew.bat
├── micronaut-cli.yml
├── settings.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── me
    │   │       └── shukawam
    │   │           ├── Application.java
    │   │           └── CountryController.java
    │   └── resources
    │       ├── application.yml # アプリケーションの設定
    │       ├── bootstrap.yml # application.yml のコンテキストを作成するために必要な設定を記述する（e.g. OCI への認証情報、OCI Vault, etc.）
    │       └── simplelogger.properties
    └── test
        ├── java
        │   └── me
        │       └── shukawam
        │           ├── CountryControllerTest.java
        │           └── CountryTest.java
        └── resources
            └── application-test.yml
```

### ATP と接続するための設定

ATP と接続するための設定を追記します。

- bootstrap.yml
  - application.yml のコンテキストを生成するために必要な設定周りを記述します
  - e.g. OCI の認証情報,OCI Vault, etc.
- application.yml
  - アプリケーションの設定を記述します
  - e.g. ATP の接続情報, etc.

#### bootstrap.yml

生成されている bootstrap.yml を以下の内容で更新します。（`.oci.vault.vaults.*`のインデントが生成時にずれているので注意してください）

```yaml
micronaut:
  application:
    name: micronautCountry
  config-client:
    enabled: true
  server:
    context-path: /sandbox # API Gateway - Deployment の Prefix Path を設定する
oci:
  vault:
    config:
      enabled: true
    vaults:
      - compartment-ocid: "<your-compartment-id>"
        ocid: "<your-vault-ocid>"
```

#### OCI Vault の Secret に必要な情報を格納する

Micronaut アプリケーションから参照するための情報を格納します。今回は、以下の情報を格納しています。

- countryDS.ocid
  - ATP の OCID
- countryDS.walletPassword
  - Wallet のダウンロード、展開に必要なパスワード
- countryDS.user
  - ATP に接続するユーザー
- countryDS.password
  - ATP に接続するユーザーのパスワード

#### application.yml

生成されている application.yml を以下の内容で更新します。
`${<secret-name>:<default-value>}`と記述すると、起動時に Secret の値に置換されます。以下、例で `<default-value>` を空にしているのは必ず Vault から参照させるためです。

```yaml
datasources:
  default:
    ocid: ${countryDS.ocid:}
    walletPassword: ${countryDS.walletPassword:}
    dialect: ORACLE
    username: ${countryDS.user:}
    password: ${countryDS.password:}
oci.config.profile: DEFAULT
jpa.default.properties.hibernate.hbm2ddl.auto: update
```

### Country.java

Entity クラスを定義します。

```Country.java
package me.shukawam.country;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Table;

@Entity(name = "Country")
@Table(name = "COUNTRY")
public class Country {
    @Id
    @Column(name = "COUNTRY_ID")
    private String countryId;
    @Column(name = "COUNTRY_NAME")
    private String countryName;

    /**
     * @return the countryId
     */
    public String getCountryId() {
        return countryId;
    }

    /**
     * @param countryId the countryId to set
     */
    public void setCountryId(String countryId) {
        this.countryId = countryId;
    }

    /**
     * @return the countryName
     */
    public String getCountryName() {
        return countryName;
    }

    /**
     * @param countryName the countryName to set
     */
    public void setCountryName(String countryName) {
        this.countryName = countryName;
    }

}

```

### CountryRepository.java

DB アクセスを行う RepositoryInterface を CrudRepository を継承して作成します。CrudRepository には、基本的な CRUD 処理を行うメソッドが生えています。

```CountryRepository.java
package me.shukawam.country;

import io.micronaut.data.annotation.Repository;
import io.micronaut.data.repository.CrudRepository;

@Repository
public interface CountryRepository extends CrudRepository<Country, String> {

}

```

### CountryService.java

今回程度の例だと冗長ですが、一応 Service も実装します。

```CountryService.java
package me.shukawam.country;

import java.util.ArrayList;
import java.util.List;

import javax.ws.rs.NotFoundException;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import jakarta.inject.Inject;
import jakarta.inject.Singleton;

@Singleton
public class CountryService {
    private final CountryRepository countryRepository;
    private static final Logger logger = LoggerFactory.getLogger(CountryService.class);

    /**
     * @param countryRepository
     */
    @Inject
    public CountryService(CountryRepository countryRepository) {
        this.countryRepository = countryRepository;
    }

    public List<Country> getAllCountry() {
        logger.debug("Inside CountryService#getAllCountry");
        List<Country> countryList = new ArrayList<>();
        countryRepository.findAll().iterator().forEachRemaining(countryList::add);
        return countryList;
    }

    public Country getCountryById(String countryId) {
        logger.debug("Inside CountryService#getCountryById");
        var optionalCountry = countryRepository.findById(countryId);
        if (optionalCountry.isEmpty()) {
            logger.info("country: %s is not found.", countryId);
            throw new NotFoundException(String.format("country: %s is not found.", countryId));
        } else {
            return optionalCountry.get();
        }
    }

}

```

### CountryController.java

REST API のエントリーポイントとなるクラスを作成します。

```CountryController.java
package me.shukawam.country;

import java.util.List;

import javax.ws.rs.core.MediaType;

import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;
import jakarta.inject.Inject;

@Controller("/country")
public class CountryController {
    private final CountryService countryService;

    /**
     * @param countryService
     */
    @Inject
    public CountryController(CountryService countryService) {
        this.countryService = countryService;
    }

    @Get
    @Produces(MediaType.APPLICATION_JSON)
    public List<Country> getAllCountry() {
        return countryService.getAllCountry();
    }

    @Get("/id/{countryId}")
    public Country getCountryById(String countryId) {
        return countryService.getCountryById(countryId);
    }

}

```

### 手元で動作確認

手元で動作確認します。InstancePrincipal が適切に設定されている OCI の VM であれば以下コマンドで動作します。(`-t` で Hot Reload がサポートされます)

```bash
./gradlew run -t
# ... omit
[main] INFO io.micronaut.runtime.Micronaut - Startup completed in 13951ms. Server Running: http://localhost:8080
```

公開されている API を叩いてみます。

```bash
curl http://localhost:8080/sandbox/country/id/JP | jq
{
  "countryId": "JP",
  "countryName": "日本国"
}
```

OK!

### Oracle Functions にデプロイ

[Micronaut on Oracle Functions ことはじめ - Oracle Functions にデプロイする](https://qiita.com/shukawam/items/27d448c8c9b269f22d3c#oracle-functions%E3%81%AB%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4%E3%81%99%E3%82%8B) をご参照ください mm

### API Gateway の設定

[Micronaut on Oracle Functions ことはじめ - API Gateway で公開する](https://qiita.com/shukawam/items/27d448c8c9b269f22d3c#api-gateway%E3%81%A7%E5%85%AC%E9%96%8B%E3%81%99%E3%82%8B) をご参照ください mm

### API Gateway + Oracle Functions の構成で動作確認

API Gateway + Oracle Functions でも同様のレスポンスが得られることが確認できます。

```bash
curl <api-gateway-host>/sandbox/country/id/JP | jq
{
  "countryId": "JP",
  "countryName": "日本国"
}
```

# 終わりに

いかがでしたでしょうか？Micronaut を使うことで Oracle Functions であっても簡単に（かつ開発体験を損ねることなく） ATP アクセスするアプリケーションを実装することができました。また、Micronaut は GraalVM による Native Image 生成と非常に相性が良いです。次回の記事では、今回作成したアプリケーションの Native Image 化にも取り組んでみたいと思います。
