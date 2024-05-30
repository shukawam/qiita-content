---
title: OpenTelemetry Operator を Java(Helidon) のアプリケーションで試してみた
tags:
  - oci
  - Helidon
  - opentelemetry
private: false
updated_at: '2023-12-18T09:38:12+09:00'
id: 24687cac7334aa256b5d
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

こちらの記事は、[Oracle Cloud Infrastructure Advent Calender 2023](https://qiita.com/advent-calendar/2023/oci) Day 18 の記事として書かれています。先日、Helidon(MicroProfile Telemetry Tracing) を用いて OpenTelemetry による手動/自動計装を実施してみました。

https://qiita.com/shukawam/items/2ea5a1aeff03901edd2b

記事中で、Kubernetes で稼働させる場合は initContainers とかで Agent をダウンロードすれば良いと思います！、と書いたのですが先日 [OpenTelemetry Operator](https://github.com/open-telemetry/opentelemetry-operator) なるものがあることを知ったので、今回はこちらを試してみます。（裏側で実施していることは結局同じですが...）

## OpenTelemetry Operator

OpenTelemetry Operator とは、OpenTelemetry Collector の管理と自動計装に対応したワークロードに対して、自動計装用の設定周りを行ってくれる Kubernetes Operator です。Instrumentation という CR で自動計装周りの設定を定義し、適切なアノテーション（`sidecar.opentelemetry.io/inject: "true"`, `instrumentation.opentelemetry.io/inject-java: "true"`, etc.）を Deployment の定義に記載すると自動計装対象のアプリケーションが稼働しているコンテナに対して、Agent のダウンロード、配置、各種環境変数の設定を自動的に実施してくれます。

詳細は、計装の手間を省きたい！OpenTelemetry に見る”自動計装”のイマというセッションが非常にわかりやすかったので、こちらを参照ください。

https://speakerdeck.com/k6s4i53rx/getting-started-auto-instrumentation-with-opentelemetry?slide=46

セッションでは、Python に関して扱っていたのですが私のほうでは Java を扱います。

### CR: Instrumentation

今回は、以下のように Instrumentation を定義してみました。同じ Kubernetes クラスタに存在する [OpenObserve](https://openobserve.ai/) を OTel のバックエンドとするように Secret の値は構成してあります。OTLP 対応のバックエンドであれば設定レベルでサクッと切り替えられるのも OpenTelemtry の良きポイントですね。

```instrumentation.yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: instrumentation
  namespace: examples
spec:
  exporter:
    endpoint: http://otel-collector:4317
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: parentbased_traceidratio
    argument: "0.25"
  java:
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
```

クラスターには、Java のアプリケーションをデプロイするため、自動計装に関する設定は Java のみを記載しており、その他は README に記載の設定例のままとなっています。

### 動きを見てみる

Instrumentation をクラスタに適用した後に、以下のような Deployment 定義のアプリケーションをデプロイします。

```deployment.yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: otel-helidon
  namespace: examples
  labels:
    app: otel-helidon
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
      annotations:
        instrumentation.opentelemetry.io/inject-java: "true"
    spec:
      containers:
        - name: otel-helidon
          image: shukawam/otel-helidon:1.0.5
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
```

これを適用し、正常にコンテナが起動されたことを確認します。

```bash
kubectl get pods | grep -i otel
otel-helidon-68885cb88-8gkp6       1/1     Running     0          34h
```

ここで、Pod の詳細を確認してみましょう。

```bash
kubectl describe pod/otel-helidon-68885cb88-8gkp6
```

```bash
Name:             otel-helidon-68885cb88-8gkp6
Namespace:        examples
Priority:         0
Service Account:  default
Node:             10.0.10.18/10.0.10.18
Start Time:       Sat, 16 Dec 2023 01:09:47 +0900
Labels:           app=otel-helidon
                  pod-template-hash=68885cb88
                  version=v1
Annotations:      instrumentation.opentelemetry.io/inject-java: true
Status:           Running
IP:               10.244.0.76
IPs:
  IP:           10.244.0.76
Controlled By:  ReplicaSet/otel-helidon-68885cb88
Init Containers:
  opentelemetry-auto-instrumentation-java:
    Container ID:  cri-o://51ea6a0053ef880b9ce0bf7808a541b74379002bb48f98e3cf42ca500494ade2
    Image:         ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:1.32.0
    Image ID:      ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java@sha256:4e5b3cbc3d89ead3bf21b271e02bf241ebbfca1bdf061f781cabcc490549bf0c
    Port:          <none>
    Host Port:     <none>
    Command:
      cp
      /javaagent.jar
      /otel-auto-instrumentation-java/javaagent.jar
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Sat, 16 Dec 2023 01:09:47 +0900
      Finished:     Sat, 16 Dec 2023 01:09:47 +0900
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  64Mi
    Requests:
      cpu:        50m
      memory:     64Mi
    Environment:  <none>
    Mounts:
      /otel-auto-instrumentation-java from opentelemetry-auto-instrumentation-java (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-d967b (ro)
Containers:
  otel-helidon:
    Container ID:   cri-o://f4ff18aaac404511ca349062814cc7b2c684033b4c886543762867e2655f5599
    Image:          shukawam/otel-helidon:1.0.5
    Image ID:       docker.io/shukawam/otel-helidon@sha256:9b7ab1a433b85d08e4d928c22ba7ef3ab659874f31f70b05527df815a00852ba
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sat, 16 Dec 2023 01:09:49 +0900
    Ready:          True
    Restart Count:  0
    Environment:
      OTEL_SERVICE_NAME:                    otel-helidon
      OTEL_TRACES_EXPORTER:                 otlp
      OTEL_METRICS_EXPORTER:                none
      OTEL_LOGS_EXPORTER:                   none
      OTEL_EXPORTER_OTLP_TRACES_PROTOCOL:   http/protobuf
      OTEL_EXPORTER_OTLP_METRICS_PROTOCOL:  http/protobuf
      OTEL_EXPORTER_OTLP_LOGS_PROTOCOL:     http/protobuf
      OTEL_EXPORTER_OTLP_TRACES_ENDPOINT:   <set to the key 'otlp-traces-endpoint' in secret 'otel-helidon-secret'>   Optional: false
      OTEL_EXPORTER_OTLP_METRICS_ENDPOINT:  <set to the key 'otlp-metrics-endpoint' in secret 'otel-helidon-secret'>  Optional: false
      OTEL_EXPORTER_OTLP_LOGS_ENDPOINT:     <set to the key 'otlp-logs-endpoint' in secret 'otel-helidon-secret'>     Optional: false
      OTEL_EXPORTER_OTLP_HEADERS:           <set to the key 'otlp-headers' in secret 'otel-helidon-secret'>           Optional: false
      OTEL_EXPORTER_OTLP_TRACES_HEADERS:    <set to the key 'otlp-headers' in secret 'otel-helidon-secret'>           Optional: false
      OTEL_EXPORTER_OTLP_METRICS_HEADERS:   <set to the key 'otlp-headers' in secret 'otel-helidon-secret'>           Optional: false
      OTEL_EXPORTER_OTLP_LOGS_HEADERS:      <set to the key 'otlp-headers' in secret 'otel-helidon-secret'>           Optional: false
      JAVA_TOOL_OPTIONS:                     -javaagent:/otel-auto-instrumentation-java/javaagent.jar
      OTEL_EXPORTER_OTLP_ENDPOINT:          http://otel-collector:4317
      OTEL_RESOURCE_ATTRIBUTES_POD_NAME:    otel-helidon-68885cb88-8gkp6 (v1:metadata.name)
      OTEL_RESOURCE_ATTRIBUTES_NODE_NAME:    (v1:spec.nodeName)
      OTEL_PROPAGATORS:                     tracecontext,baggage,b3
      OTEL_TRACES_SAMPLER:                  parentbased_traceidratio
      OTEL_TRACES_SAMPLER_ARG:              1.0
      OTEL_RESOURCE_ATTRIBUTES:             k8s.container.name=otel-helidon,k8s.deployment.name=otel-helidon,k8s.namespace.name=examples,k8s.node.name=$(OTEL_RESOURCE_ATTRIBUTES_NODE_NAME),k8s.pod.name=$(OTEL_RESOURCE_ATTRIBUTES_POD_NAME),k8s.replicaset.name=otel-helidon-68885cb88,service.version=1.0.5
    Mounts:
      /otel-auto-instrumentation-java from opentelemetry-auto-instrumentation-java (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-d967b (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-d967b:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
  opentelemetry-auto-instrumentation-java:
    Type:        EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:   200Mi
QoS Class:       Burstable
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:          <none>
```

このように Deployment の環境変数に OpenTelemetry 関係の設定を何も記載していなくても `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`, `OTEL_EXPORTER_OTLP_HEADERS` 等の環境変数が設定されていることがわかります。
また、initContainers に opentelemetry-auto-instrumentation-java というコンテナが追加されていることが確認でき、`cp /javaagent.jar /otel-auto-instrumentation-java/javaagent.jar` からもわかる通り、OpenTelemetry の自動計装を行う Java の Agent を `opentelemetry-auto-instrumentation-java` のマウント先である `/otel-auto-instrumentation-java` にコピーし、otel-helidon というコンテナでは同じボリューム（`opentelemetry-auto-instrumentation-java`）を `otel-auto-instrumentation-java` にマウントしています。そのため、環境変数である `JAVA_TOOL_OPTIONS` では `-javaagent:/otel-auto-instrumentation-java/javaagent.jar`　という Agent のパスが指定されています。

いくつか、エンドポイントを実行してみるときちんとトレーシング情報が見れることが確認できます。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/24d91d1e-ac24-20da-511f-8f85ac85e982.png)

![image02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/0005eae9-9116-30ad-9ac9-fb57cb6c7e22.png)



# 終わりに

今回は、Kubernetes 上の Java アプリケーションに対して、OpenTelemetry による自動計装をより簡単に行う方法を実施してみました。
