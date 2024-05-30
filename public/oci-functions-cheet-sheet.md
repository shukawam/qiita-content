---
title: OCI Functions Cheat Sheet
tags:
  - oci
private: false
updated_at: '2023-08-15T14:59:42+09:00'
id: 02db794cb710037594e5
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

これどうするんだっけ？となりドキュメントを見に行くことが（個人的に）多い OCI Functions 開発におけるチートシートです。内容は順次拡張します。

## Fn Test Harness

### Request Header/Body をエミュレートしたい

詳細は、[FDK Java - Testing data binding and configuration](https://github.com/fnproject/fdk-java/blob/master/docs/TestingFunctions.md#testing-data-binding-and-configuration) から。

#### 単純な文字列を受け取る

以下のような Function で input を受け取った場合のテストをしたいとします。

```HelloFunction.java
public class HelloFunction {

    public String handleRequest(String input) {
        String name = (input == null || input.isEmpty()) ? "world"  : input;

        System.out.println("Inside Java Hello World function");
        return "Hello, " + name + "!";
    }

}
```

→ `FnEventBuilderJUnit4#withBody`, `FnEventBuilderJUnit4#withHeader`を使う

Request Body に関わらず、Event をエミュレートするためのインターフェースが `com.fnproject.fn.testing.FnEventBuilder` から提供されています。
`FnEventBuilderJUnit4` はこのインターフェースの実装クラスです。

```HelloFunctionTest.java
import com.fnproject.fn.testing.*;
import org.junit.*;

import static org.junit.Assert.*;

public class HelloFunctionTest {

    @Rule
    public final FnTestingRule testing = FnTestingRule.createDefault();

    @Test
    public void shouldReturnGreetingWithBody() {
        testing.givenEvent().withBody("John").enqueue();
        testing.thenRun(HelloFunction.class, "handleRequest");

        FnResult result = testing.getOnlyResult();
        assertEquals("Hello, John!", result.getBodyAsString());
    }

}
```

#### JSON を受け取る

以下のような JSON を受け取る Function のテストをしたいとします。

```HelloJsonFunction.java
public class HelloJsonFunction {

    public String handleRequest(Input input) {
        System.out.println("Inside Java Hello World function");
        return "Hello, " + input.name() + "!";
    }

}
```

```Input.java
public record Input(String name) {
}
```

FDK(Function Development Kits)には、Jackson への依存が含まれているので、以下のように実装すると良いでしょう。

```HelloJsonFunctionTest.java
import static org.junit.Assert.assertEquals;

import org.junit.Rule;
import org.junit.Test;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fnproject.fn.testing.FnResult;
import com.fnproject.fn.testing.FnTestingRule;

public class HelloJsonFunctionTest {
    @Rule
    public final FnTestingRule testing = FnTestingRule.createDefault();

    @Test
    public void shouldReturnGreetingWithBody() {
        var mapper = new ObjectMapper();
        try {
            testing
                .givenEvent()
                .withHeader("content-type", "application/json") // Request Header
                .withBody(mapper.writeValueAsString(new Input("John"))) // Request Body
                .enqueue();
            testing.thenRun(HelloJsonFunction.class, "handleRequest");
        } catch (JsonProcessingException e) {
            // do something.
        }
        FnResult result = testing.getOnlyResult();
        assertEquals("Hello, John!", result.getBodyAsString());
    }
}
```

#### Config を使いたい

OCID(OCI のリソースに付与された ID)やデータベースの接続先（を参照する Vault の OCID）などは、Config に持っておくケースがあると思います。

```func.yaml
schema_version: 20180708
name: test-function
version: 0.0.1
runtime: java
build_image: fnproject/fn-java-fdk-build:jdk17-1.0.177
run_image: fnproject/fn-java-fdk:jre17-1.0.177
cmd: com.example.fn.HelloFunction::handleRequest
config:
    greet: Hi
```

`@FnConfiguration` を用いるパターン

```HelloConfigFunction.java
import com.fnproject.fn.api.FnConfiguration;
import com.fnproject.fn.api.RuntimeContext;

public class HelloConfigFunction {
    private String greet;

    @FnConfiguration
    public void config(RuntimeContext ctx) {
        greet = ctx.getConfigurationByKey("greet").orElse("Hello");
    }

    public String handleRequest(String input) {
        String name = (input == null || input.isEmpty()) ? "world"  : input;

        System.out.println("Inside Java Hello World function");
        return greet + name + "!";
    }
}
```

final + constructor でも実装出来ますが、Function の実行毎に呼びだされるので、Runtime 毎に 1 回（Warm Start として使いまわされる単位で 1 回）だけ実行したい場合は、素直に `@FnConfiguration` を使うと良いでしょう。（例えば、データベースへのコネクション定義なんかは、`@FnConfiguration` を付けたメソッド内で行うのが良いです。）

```HelloConstructorConfigFunction.java
import com.fnproject.fn.api.RuntimeContext;

public class HelloConstructorConfigFunction {
    private final String greet;

    public HelloConstructorConfigFunction(RuntimeContext ctx) {
        greet = ctx.getConfigurationByKey("GREET").orElse("Hello");
    }

    public String handleRequest(String input) {
        String name = (input == null || input.isEmpty()) ? "world" : input;

        System.out.println("Inside Java Hello World function");
        return greet + ", " + name + "!";
    }
}
```

テストは以下のように書くことができます。

```HelloConfigFunctionTest.java
import static org.junit.Assert.assertEquals;

import org.junit.Rule;
import org.junit.Test;

import com.fnproject.fn.testing.FnResult;
import com.fnproject.fn.testing.FnTestingRule;

public class HelloConfigFunctionTest {

    @Rule
    public final FnTestingRule testing = FnTestingRule.createDefault();

    @Test
    public void shouldReturnDefaultGreeting() {
        testing.givenEvent().enqueue();
        testing.thenRun(HelloConfigFunction.class, "handleRequest");

        FnResult result = testing.getOnlyResult();
        assertEquals("Hello, world!", result.getBodyAsString());
    }

    @Test
    public void shouldReturnGreetingWithConfig() {
        testing.setConfig("greet", "Hi").givenEvent().withBody("John").enqueue();
        testing.thenRun(HelloConfigFunction.class, "handleRequest");

        FnResult result = testing.getOnlyResult();
        assertEquals("Hi, John!", result.getBodyAsString());
    }

    @Test
    public void shouldReturnGreetingWithConstructorConfig() {
        testing.setConfig("greet", "Hi").givenEvent().withBody("John").enqueue();
        testing.thenRun(HelloConstructorConfigFunction.class, "handleRequest");

        FnResult result = testing.getOnlyResult();
        assertEquals("Hi, John!", result.getBodyAsString());
    }

}
```

`FnTestingRule#setConfig` や Config ファイルから渡した Config 名（ここでいう、"greet"）は大文字に置換され、`-` が含まれている場合は `_` に置換されることに注意が必要です。そのため、渡された Configを扱う箇所は、`greet = ctx.getConfigurationByKey("GREET").orElse("Hello")` と実装しています。

#### 同じイベントを複数回送信してテストをしたい

→ `FnEventBuilder#enqueue(int n)` を使うと、イベントのコピーを指定回数分（n）送ることができる

```HelloFunctionTest.java
import com.fnproject.fn.testing.*;
import org.junit.*;

import static org.junit.Assert.*;

import java.util.List;

public class HelloFunctionTest {

    @Rule
    public final FnTestingRule testing = FnTestingRule.createDefault();

    @Test
    public void shouldReturnSameGreeting() {
        testing.givenEvent().enqueue(10);
        testing.thenRun(HelloFunction.class, "handleRequest");

        List<FnResult> results = testing.getResults();
        results.forEach(result -> {
            assertEquals("Hello, world!", result.getBodyAsString());
        });
    }

}
```

結果を受け取る際は、`FnTestingRule#getResults` を用いると、実行結果が `List<FnResult>` で受け取れる。

# 参考

https://github.com/fnproject/fdk-java

