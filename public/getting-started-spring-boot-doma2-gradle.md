---
title: 'Spring Boot, Doma2, Gradleの初期設定まとめ'
tags:
  - Java
  - Eclipse
  - gradle
  - SpringBoot
  - doma2
private: false
updated_at: '2022-01-04T21:24:22+09:00'
id: ca3db198926b1b484c15
organization_url_name: null
slide: false
ignorePublish: false
---
# 始めに

Eclipse で開発する際の、Spring Boot + Doma2 + Gradle の初期設定でいつもハマっているので自分用にまとめる。



# 環境

- IDE：Eclipse pleiades-2020-03
- Spring Boot：2.3.1 RELEASE
- Doma：2.35.0


# 設定手順

## （省略可）Spring Initializer でアプリケーションのひな型を作成する

Eclipse のプラグイン経由で作成していますが、ブラウザでも curl でもなんでもいいです。

![image-01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/eb24c07a-7724-e4f0-0de7-3494bca82607.png)

![image-02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/aa74946f-67ea-cbc5-85eb-405a029500a2.png)

## Doma2 への依存関係を追加する

```build.gradle
dependencies {
  // ... 省略
  implementation 'org.seasar.doma.boot:doma-spring-boot-starter:1.4.0'
  annotationProcessor 'org.seasar.doma:doma-processor:2.35.0'
}
```

## Eclipse の設定をする

`build.gradle`の plugins に Eclipse 設定用のプラグインを追加する。

```build.gradle
plugins {
  // ...
  id 'com.diffplug.eclipse.apt' version '3.23.0'
}
```

`gradle eclipse`を実行し、設定を反映する。

```
$ ./gradlew eclipse
Starting a Gradle Daemon (subsequent builds will be faster)

BUILD SUCCESSFUL in 16s
5 actionable tasks: 5 executed
```

Eclipse の設定（注釈処理）が完了していることが確認できる。

![image-03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/ee74cbfc-1aba-ab7b-4ac5-f4ace1959735.png)

![image-04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/f34dfecb-15f9-67ed-12ca-a2da3ac5ea62.png)

# 簡単なAPIを作成する

[https://github.com/domaframework/doma-spring-boot/tree/1.4.0](https://github.com/domaframework/doma-spring-boot/tree/1.4.0)の README に沿って、基本的に作成します。

## Entityを定義する

```Reservation.java
package com.example.demo;

import org.seasar.doma.Entity;

import lombok.Data;

@Data
@Entity
public class Reservation {

	private Integer id;

	private String name;
}
```



## Dao Interface を定義する

```ReservationDao.java
package com.example.demo;

import java.util.List;

import org.seasar.doma.Dao;
import org.seasar.doma.Insert;
import org.seasar.doma.Select;
import org.seasar.doma.boot.ConfigAutowireable;
import org.springframework.transaction.annotation.Transactional;

@ConfigAutowireable
@Dao
public interface ReservationDao {

	@Select
	public List<Reservation> selectAll();

	@Insert
	@Transactional
	public int insert(Reservation reservation);
}
```

`ReservationDao#selectAll`にカーソルを合わせ、**右クリック** > **Doma** > **Jump to Sql File** を押下すると、空のSQL ファイルが生成されます。（Eclipse に Doma Tools プラグインを追加している場合）

※ DOMA4019 が発生した場合は、[（補足）DOMA4019エラーが発生した場合](# DOMA4019エラーが発生した場合)を参照してください。

## SQL ファイルにクエリーを書く

```selectAll.sql
SELECT
  id,
  name
FROM reservation
ORDER BY name ASC
```

## Service, Controller クラスを定義する

```ReservationService.java
package com.example.demo;

import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class ReservationService {

	@Autowired
	private ReservationDao reservationDao;

	public List<Reservation> selectAll() {
		return reservationDao.selectAll();
	}

	public int insert(Reservation reservation) {
		return reservationDao.insert(reservation);
	}
}

```

```ReservationController.java
package com.example.demo;

import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ReservationController {

	@Autowired
	private ReservationService reservationService;

	@GetMapping(path = "/")
	public List<Reservation> selectAll() {
		return reservationService.selectAll();
	}

	@PostMapping(path = "/")
	public int insert(@RequestBody Reservation reservation) {
		return reservationService.insert(reservation);
	}

}
```



## 起動時に`Reservation`テーブルを作成する

[HSQLDB](http://hsqldb.org/)という Java 製のインメモリ DB を使います。`src/main/resources`直下に`schema.sql`を配置しておくと、起動時にテーブル作成のスクリプトを流すことができます。

```schema.sql
CREATE TABLE reservation (
  id   IDENTITY,
  NAME VARCHAR(50)
);
```

# APIの動作確認

POST（登録）

```http
POST http://localhost:8080 HTTP/1.1
Content-Type: application/json

{
  "id": 1,
  "name": "サンプルA"
}
```

レスポンス；

```http
HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Sun, 12 Jul 2020 15:09:30 GMT
Connection: close

1
```

GET（全件取得）

```http
GET http://localhost:8080 HTTP/1.1
```

レスポンス；

```http
HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Sun, 12 Jul 2020 15:10:20 GMT
Connection: close

[
  {
    "id": 1,
    "name": "サンプルA"
  }
]
```

# （補足）DOMA4019 エラーが発生した場合

以下のように、SQL ファイルの絶対パスが期待通りではない、と DOMA4019 エラーが発生した場合について。

```
[DOMA4019] The file "META-INF/com/example/demo/ReservationDao/selectAll.sql" is not found in the classpath. The absolute path is "C:\Git\springboot-doma2-sample\bin\main\META-INF\com\example\demo\ReservationDao\selectAll.sql".
```

対象プロジェクト上で、**右クリック** > **プロパティ** > **Java のビルドパス** を修正します。

![image-05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/1a8bc73c-ba57-eecc-e564-55d7222496f2.png)

プロジェクトのデフォルト出力フォルダー ⇒ 特定の出力フォルダー（`bin/main`）に修正。

![image-06-01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/5087f9dd-3b59-94c2-4af9-c54c12b2cb5f.png)

# 終わりに

作成したアプリケーションは、[リポジトリ](https://gitlab.com/s.kawamura/springboot-doma2-sample)に格納しました。`build.gradle`の全量を参照したい場合などに見てください。

# 参考

- [https://doma.readthedocs.io/en/2.35.0/getting-started-eclipse/](https://doma.readthedocs.io/en/2.35.0/getting-started-eclipse/)
- [https://doma.readthedocs.io/en/2.35.0/build/#build-with-eclipse](https://doma.readthedocs.io/en/2.35.0/build/#build-with-eclipse)
- [https://github.com/domaframework/doma-spring-boot/tree/1.4.0](https://github.com/domaframework/doma-spring-boot/tree/1.4.0)

