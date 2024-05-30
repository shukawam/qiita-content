---
title: Helidon を用いた MicroProfile Telemetry Tracing
tags:
  - oci
  - microprofile
  - Helidon
  - opentelemetry
private: false
updated_at: '2024-01-05T17:20:06+09:00'
id: 2ea5a1aeff03901edd2b
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

こちらの記事は、[Oracle Cloud Infrastructure Advent Calender 2023](https://qiita.com/advent-calendar/2023/oci) Day 4 の記事として書かれています。

2023/10 に Java のマイクロサービス開発用のフレームワークである Helidon は 4.0.0 がリリースされ、MicroProfile 6.0 に対応しました。それ以外にも Web サーバーの実装が Netty から Virtual Threads ベースの Níma に置き換わったりと大きく変化をしています。

https://github.com/helidon-io/helidon/releases/tag/4.0.0

MicroProfile 6.0 の文脈で言えば、近年の分散トレーシングの標準化に伴い、MicroProfile Telemetry Tracing という新しい仕様が追加されています。これは、OpenTelemetry 1.13 仕様に準拠するもので、MicroProfile アプリケーションが OpenTelemetry（以下、OTel） を介して分散トレーシングが有効になっている環境に簡単に参加できるようにするための振舞いが定義されています。ただし、仕様にも記載のある通り、OTel の Metrics, Logging の統合は、MicroProfile Telemetry Tracing の範囲外となっており、実装者は必要に応じて Metrics, Logging の両方のサポートを自由に実装を提供することができます。

そこで今回は、MicrpProfile 仕様を実装したフレームワークである Helidon を用いて MicroProfile Telemetry Tracing を試してみようと思います。OTel のバックエンドは、Oracle Cloud Infrastructure の 通称 APM（Application Performance Monitoring）というサービスを使用します。

## 全体の構成図

以下のように、Kubernetes(OKE)上で Helidon を用いたアプリケーションを稼働させ、OTLP(OpenTelemetry Protocol)を用いて、テレメトリーデータを APM へ送信します。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/0ed13184-de37-662e-71d1-cf1898610928.png)

尚、使用するサンプルアプリケーションは作成済みの以下のアプリケーションを使用します。

https://github.com/shukawam/otel-examples/tree/main/otel-helidon

エンドポイントは、2 つ存在し、それぞれ以下のように振舞います。

- `/api/cowsay/say`: MicroProfile REST Client を用いて、`/remote/cowsay/say` にリクエストを送信
- `/api/cowsay/delay`: MicroProfile REST Client を用いて、`/remote/cowsay/delay` にリクエストを送信

## Automatic Instrumentation

MicroProfile に限らず、Java の自動計装は専用の Java Agent を実行中の JVM にアタッチするだけでテレメトリーデータの収集が可能です。簡単で良きですね！
コンテナで実行する場合は、Dockerfile を以下のように書いておけば環境によって変わり得る部分は、環境変数から読み込む等の工夫がしやすいと思います。（ ~~Kubernetes 上で稼働させる場合は、initContainers などで Agent の JAR ファイルを取得するのも良いかもしれません。~~ ）

OpenTelemetry Operator の存在をこの記事を執筆した後に知ったので、合わせて記事としました。実施していることは、initContainers を用いた方法と同じですが、既に存在する Operator があるのでこちらを使うのがよろしいかと思います。

https://qiita.com/shukawam/items/24687cac7334aa256b5d

```dockerfile
FROM maven:3.9.5 as build
WORKDIR /helidon
ADD pom.xml pom.xml
RUN mvn package -Dmaven.test.skip -Declipselink.weave.skip -Declipselink.weave.skip -DskipOpenApiGenerate
ADD src src
RUN mvn package -DskipTests
RUN echo "done!"

# Multistage build の 1 stage で使用する OpenTelemetry Java Agent をダウンロードする
FROM busybox as download
WORKDIR /var/cache
RUN wget https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v1.32.0/opentelemetry-javaagent.jar
RUN echo "done! - download stage"

FROM eclipse-temurin:21-jre
WORKDIR /helidon
COPY --from=build /helidon/target/otel-helidon.jar ./
COPY --from=build /helidon/target/libs ./libs
COPY --from=download /var/cache ./agent
ENV JAVA_TOOL_OPTIONS=-javaagent:/helidon/agent/opentelemetry-javaagent.jar
CMD ["java", "-jar", "otel-helidon.jar"]
EXPOSE 8080
```

上記のように `JAVA_TOOL_OPTIONS` を設定しても良いですし、`java -javaagent=<path-to-otel-javaagent>.oteltelemetry-javaagent.jar -jar xxx.jar` のように起動時にパラメータとして渡してあげても良いです。

これを Kubernetes 上で稼働させるための Deployment の定義は以下のようになっています。

```deployment.yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: otel-helidon
  namespace: examples
spec:
  replicas: 1
  selector:
    matchLabels:
      app: otel-helidon
  template:
    metadata:
      labels:
        app: otel-helidon
        version: v1
    spec:
      containers:
        - name: otel-helidon
          image: shukawam/otel-helidon:1.0.4
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          env:
            - name: OTEL_SERVICE_NAME
              value: otel-helidon
            - name: OTEL_TRACES_EXPORTER
              value: otlp
            - name: OTEL_METRICS_EXPORTER
              value: none
            - name: OTEL_LOGS_EXPORTER
              value: none
            - name: OTEL_EXPORTER_OTLP_TRACES_PROTOCOL
              value: http/protobuf
            - name: OTEL_EXPORTER_OTLP_METRICS_PROTOCOL
              value: http/protobuf
            - name: OTEL_EXPORTER_OTLP_LOGS_PROTOCOL
              value: http/protobuf
            - name: OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
              valueFrom:
                secretKeyRef:
                  key: otlp-traces-endpoint
                  name: otel-helidon-secret
            - name: OTEL_EXPORTER_OTLP_METRICS_ENDPOINT
              valueFrom:
                secretKeyRef:
                  key: otlp-metrics-endpoint
                  name: otel-helidon-secret
            # - name: OTEL_EXPORTER_OTLP_LOGS_ENDPOINT
            #   valueFrom:
            #     secretKeyRef:
            #       key: otlp-logs-endpoint
            #       name: otel-helidon-secret
            - name: OTEL_EXPORTER_OTLP_HEADERS
              valueFrom:
                secretKeyRef:
                  key: otlp-headers
                  name: otel-helidon-secret
            - name: OTEL_EXPORTER_OTLP_TRACES_HEADERS
              valueFrom:
                secretKeyRef:
                  key: otlp-headers
                  name: otel-helidon-secret
            - name: OTEL_EXPORTER_OTLP_METRICS_HEADERS
              valueFrom:
                secretKeyRef:
                  key: otlp-headers
                  name: otel-helidon-secret
            # - name: OTEL_EXPORTER_OTLP_LOGS_HEADERS
            #   valueFrom:
            #     secretKeyRef:
            #       key: otlp-headers
            #       name: otel-helidon-secret
```

OTLP exporter の設定は、Java system properties or 環境変数から行うことができますが今回は環境変数から行っています。

```yaml
env:
  - name: OTEL_SERVICE_NAME
    value: otel-helidon
  - name: OTEL_TRACES_EXPORTER
    value: otlp
  - name: OTEL_METRICS_EXPORTER
    value: none
  - name: OTEL_LOGS_EXPORTER
    value: none
  - name: OTEL_EXPORTER_OTLP_TRACES_PROTOCOL
    value: http/protobuf
  - name: OTEL_EXPORTER_OTLP_METRICS_PROTOCOL
    value: http/protobuf
  - name: OTEL_EXPORTER_OTLP_LOGS_PROTOCOL
    value: http/protobuf
  - name: OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
    valueFrom:
      secretKeyRef:
        key: otlp-traces-endpoint
        name: otel-helidon-secret
  #   - name: OTEL_EXPORTER_OTLP_METRICS_ENDPOINT
  #     valueFrom:
  #       secretKeyRef:
  #         key: otlp-metrics-endpoint
  #         name: otel-helidon-secret
  # - name: OTEL_EXPORTER_OTLP_LOGS_ENDPOINT
  #   valueFrom:
  #     secretKeyRef:
  #       key: otlp-logs-endpoint
  #       name: otel-helidon-secret
  - name: OTEL_EXPORTER_OTLP_HEADERS
    valueFrom:
      secretKeyRef:
        key: otlp-headers
        name: otel-helidon-secret
  - name: OTEL_EXPORTER_OTLP_TRACES_HEADERS
    valueFrom:
      secretKeyRef:
        key: otlp-headers
        name: otel-helidon-secret
#   - name: OTEL_EXPORTER_OTLP_METRICS_HEADERS
#     valueFrom:
#       secretKeyRef:
#         key: otlp-headers
#         name: otel-helidon-secret
# - name: OTEL_EXPORTER_OTLP_LOGS_HEADERS
#   valueFrom:
#     secretKeyRef:
#       key: otlp-headers
#       name: otel-helidon-secret
```

`OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`, `OTEL_EXPORTER_OTLP_HEADERS` に何を設定すれば良いか？は、[APM のドキュメント](https://docs.oracle.com/ja-jp/iaas/application-performance-monitoring/doc/configure-open-source-tracing-systems.html)に記載があります。

- `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`: `https://<dataUploadEndpoint>/20200101/opentelemetry/private/v1/traces`
- `OTEL_EXPORTER_OTLP_HEADERS`: `Authorization=dataKey <your-datakey>`

コンソールだと以下部分から参照すれば良いです。

![image02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/a040134a-5da7-e771-6682-1b4a49c129d4.png)

実際に動かして `/api/cowsay/say` にリクエストを送ってみると、以下のように APM にテレメトリーデータが収集されていることがわかります。

![image03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/60db39ae-0bbb-0a0d-0939-8d59f0ce2f38.png)

## Manual Instrumentation

自動計装で不十分な場合は、明示的にスパン情報を追加できます。スパンを追加するには、CDI ベースで測定したいメソッドに `@WithSpan` というアノテーションをつけるか、`io.opentelemetry.api.trace.Tracer.spanBuilder` を使って、スパンの開始と終了を宣言すれば良いです。

`@WithSpan` を用いる例

```java
@WithSpan(value = "cdi.span")
private void delayWithCDISpan() {
    delayWithSpanBuilder();
    try {
        Thread.sleep(DELAY);
    } catch (InterruptedException | IllegalArgumentException e) {
        throw new RuntimeException(e);
    }
}
```

`io.opentelemetry.api.trace.Tracer.spanBuilder` を用いる例

```java
import java.util.Optional;
import java.util.logging.Level;
import java.util.logging.Logger;

import com.github.ricksbrown.cowsay.Cowsay;

import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Context;
import io.opentelemetry.instrumentation.annotations.WithSpan;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.QueryParam;

@Path("/remote/cowsay")
public class CowsayRemoteExecutor {
    private final Tracer tracer;

    @Inject
    public CowsayRemoteExecutor(Tracer tracer) {
        this.tracer = tracer;
    }
    // ...

    private void delayWithSpanBuilder() {
        var span = tracer.spanBuilder("builder.span")
                .setSpanKind(SpanKind.INTERNAL)
                .setParent(Context.current())
                .setAttribute("my.attribute","value")
                .startSpan();
        try {
            Thread.sleep(DELAY);
        } catch (InterruptedException | IllegalArgumentException e) {
            throw new RuntimeException(e);
        }
        span.end();
    }
}
```

`io.opentelemetry.api.trace.Tracer.spanBuilder` を用いた場合には、独自の属性が追加できたり、スパンの開始時刻を設定できたりとスパンの生成をより柔軟に行うことができます。

使用したサンプルアプリケーションの `/api/cowsay/delay` の方には、CDI ベースと spanBuilder を使うパターンでカスタムスパンを 2 つ仕込んでいます。実際に、エンドポイントを実行し APM を参照してみると以下のようになっています。

![image04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/bf30241d-2ddc-e3c0-1499-987c02bba9db.png)

きちんとトレーシングの情報を参照することができました。

# 終わりに

今回は、Helidon 4.0.0 で使えるようになった MicroProfile 6.0 に含まれている MicroProfile Telemetry Tracing を試してみました。OpenTelemetry のような言語非依存な仕様を活用することで Cloud Native な環境下におけるテレメトリー収集をいい感じに効率よく（大事！）行えると思います。

# 参考

https://download.eclipse.org/microprofile/microprofile-telemetry-1.0/tracing/microprofile-telemetry-tracing-spec-1.0.html
