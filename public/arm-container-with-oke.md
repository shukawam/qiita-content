---
title: OKEでARMコンテナを動かす
tags:
  - ARM
  - kubernetes
  - oci
  - oraclecloud
  - OKE
private: false
updated_at: '2021-12-14T12:59:10+09:00'
id: 22f728b57763ea8030aa
organization_url_name: oracle
slide: false
ignorePublish: false
---
# 始めに

5 月の終わりごろから 6 月に掛けて、Twitter などでちょっと話題になっていた Oracle Cloud Infrastructure(OCI) のニュースを覚えていますでしょうか？

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/4ba50493-d307-500b-43aa-89ef7787b801.png)

※[https://www.oracle.com/jp/corporate/pressrelease/jp20210526.html](https://www.oracle.com/jp/corporate/pressrelease/jp20210526.html) より引用

上にもあるように、ARM ベースのインスタンスが [Always Free](https://www.oracle.com/jp/cloud/free/)で合計 4 コア、24GB メモリまで永久に無償で使えるというものです。現在、東京リージョンはリソースがひっ迫しているので、新しくアカウントを作成する方はホームリージョンを大阪や Ashburn にすることをお勧めします。(Ashburn リージョンでは普通に作れました) また、それに関連して[Oracle Container Engine for Kubernetes（OKE）の Arm インスタンスのサポート](https://blogs.oracle.com/oracle4engineer/oke-arm-support)というブログも公開されています。
そこで今回は、そんな ARM ベースの Ampere A1Compute と AMD ベースのインスタンスで OKE[^1]の Node Pool を構築し、実際にコンテナアプリケーションが動作するところまでを確認してみたいと思います。

[^1]: Oracle Container Engine for Kubernetes: OCI が提供するマネージド Kubernetes サービス

# 環境のゴール

最終的に以下のような環境を作成します。

![architecture.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/9a8199f2-4f2b-a738-6de1-ceec9c92434b.png)

OKE で管理する Node Pool を、ARM ベースの Ampere A1Compute と AMD ベースのインスタンスで構築しそれらが混在するような構成とします。

# 手順

## マルチアーキテクチャサポート

Docker Hub を使っている方にとってはお馴染みかと思いますが、コンテナレジストリで複数の CPU アーキテクチャの Docker イメージを格納しておくと、イメージの取得時に OS と CPU アーキテクチャに対応するイメージを自動的(もしくは手動)に取得してくれる機能です。今回は、[Helidon](https://oracle-japan-oss-docs.github.io/helidon/docs/v2/#/about/01_overview)[^2] で実装したコンテナアプリケーションをマルチ CPU アーキテクチャに対応させたいと思います。

[^2]: Java ベースの軽量フレームワーク

まずは、アプリケーションを作成します。今回は、単純に"Hello World"と返すシンプルな API を作ります。

```bash
helidon init --flavor MP --archetype bare --groupid com.example --package com.example.greet --name greet
```

実行すると以下のようなファイル等が生成されます。

```bash
C:.
│  .helidon
│  pom.xml
│  README.md
│
└─src
    ├─main
    │  ├─java
    │  │  └─com
    │  │      └─example
    │  │          └─greet
    │  │                  GreetResource.java
    │  │                  package-info.java
    │  │
    │  └─resources
    │      │  logging.properties
    │      │
    │      └─META-INF
    │          │  beans.xml
    │          │  microprofile-config.properties
    │          │
    │          └─native-image
    │                  reflect-config.json
    │
    └─test
        └─java
            └─com
                └─example
                    └─greet
                            MainTest.java
```

まずは、ローカルで実行してみます。

```bash
# build the application
helidon build
# run the application
java -jar .\target\greet.jar
2021.07.20 11:52:25 情報 io.helidon.common.LogConfig Thread[main,5,main]: Logging at initialization configured using classpath: /logging.properties
2021.07.20 11:52:27 情報 io.helidon.tracing.tracerresolver.TracerResolverBuilder Thread[main,5,main]: TracerResolver not configured, tracing is disabled
2021.07.20 11:52:27 警告 io.helidon.microprofile.tracing.TracingCdiExtension Thread[main,5,main]: helidon-microprofile-tracing is
on the classpath, yet there is no tracer implementation library. Tracing uses a no-op tracer. As a result, no tracing will be configured for WebServer and JAX-RS
2021.07.20 11:52:27 情報 io.helidon.microprofile.security.SecurityCdiExtension Thread[main,5,main]: Authentication provider is missing from security configuration, but security extension for microprofile is enabled (requires providers configuration at key security.providers). Security will not have any valid authentication provider
2021.07.20 11:52:27 情報 io.helidon.microprofile.security.SecurityCdiExtension Thread[main,5,main]: Authorization provider is missing from security configuration, but security extension for microprofile is enabled (requires providers configuration at key security.providers). ABAC provider is configured for authorization.
2021.07.20 11:52:27 情報 io.helidon.microprofile.server.ServerCdiExtension Thread[main,5,main]: Registering JAX-RS Application: HelidonMP
2021.07.20 11:52:28 情報 io.helidon.webserver.NettyWebServer Thread[nioEventLoopGroup-2-1,10,main]: Channel '@default' started: [id: 0x24ea6c3f, L:/[0:0:0:0:0:0:0:0]:8080]
2021.07.20 11:52:28 情報 io.helidon.microprofile.server.ServerCdiExtension Thread[main,5,main]: Server started on http://localhost:8080 (and all other host addresses) in 2412 milliseconds (since JVM startup).
2021.07.20 11:52:28 情報 io.helidon.common.HelidonFeatures Thread[features-thread,5,main]: Helidon MP 2.3.2 features: [CDI, Config, Fault Tolerance, Health, JAX-RS, Metrics, Open API, REST Client, Security, Server, Tracing]
2021.07.20 11:52:36 情報 io.helidon.webserver.NettyWebServer Thread[nioEventLoopGroup-2-1,10,main]: Channel '@default' closed: [id: 0x24ea6c3f, L:/[0:0:0:0:0:0:0:0]:8080]
2021.07.20 11:52:36 情報 io.helidon.microprofile.server.ServerCdiExtension Thread[helidon-cdi-shutdown-hook,5,main]: Server stopped in 12 milliseconds.
```

公開されているエンドポイントを叩いてみます。

```bash
curl http://localhost:8080/greet
{"message":"Hello World!"}
```

次に、コンテナアプリケーションとして動作させるために Dockerfile を以下のように作成します。(マルチステージビルドを使用してコンテナイメージを作成しています)

```Dockerfile
# 1st stage, build the app
FROM maven:3.6-jdk-11 as build

WORKDIR /helidon

# Create a first layer to cache the "Maven World" in the local repository.
# Incremental docker builds will always resume after that, unless you update
# the pom
ADD pom.xml .
RUN mvn package -Dmaven.test.skip -Declipselink.weave.skip

# Do the Maven build!
# Incremental docker builds will resume here when you change sources
ADD src src
RUN mvn package -DskipTests

RUN echo "done!"

# 2nd stage, build the runtime image
FROM openjdk:11-jre-slim
WORKDIR /helidon

# Copy the binary built in the 1st stage
COPY --from=build /helidon/target/greet.jar ./
COPY --from=build /helidon/target/libs ./libs

CMD ["java", "-jar", "greet.jar"]

EXPOSE 8080
```

次に [Docker Buildx](https://matsuand.github.io/docs.docker.jp.onthefly/buildx/working-with-buildx/) を使用してマルチプラットフォームのイメージビルドを行います。今回は、AMD, ARM の両方に対応させるためそのためのイメージをビルドします。

### Docker Hub

Docker イメージのビルド時に Docker Hub に対して自動的にイメージの push を行いたいので、あらかじめログインしておきます。

```bash
# login docker hub
docker login
Authenticating with existing credentials...
Login Succeeded
```

マルチプラットフォーム(linux/amd64, linux/arm64)のビルドを行います。

```bash
# create a container for building with linux/ARM64
docker buildx create --use --name ARM-build --platform linux/ARM64,linux/amd64
# build a container
docker buildx build --platform linux/amd64,linux/ARM64 . --push
# push a docker image
```

Docker Hub を確認してみると、確かに作成した Docker イメージに対して複数アーキテクチャがサポートされていることが分かります。

![image03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/98daf0c4-73e7-d84c-4e15-a45ceebcaf71.png)

これで、AMD, ARM のインスタンスに対して、OS/アーキテクチャを判断し自動的に適切なイメージを取得することができるようになります。

### OCIR[^3]

[^3]: Oracle Cloud Infrastructure Registry: OCI が提供する Docker v2 対応のコンテナレジストリサービス

実際の開発では、CI/CD パイプラインの中で Docker イメージをビルドするケースが多いと思うのでそっちのパターンにも簡単に触れておきます。今回は、コンテナレジストリで OCIR を使用しますが Docker Hub でも同じです。以下、ざっくりとした全体構成です。

![image04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/bffbec96-195d-50d5-07c9-157fae4b3e36.png)

1. ソースコードの変更をリポジトリ(GitHub)に対して push する
2. ソースコードの変更を検知し、CI/CD パイプラインが実行される
3. コンテナイメージのビルド、コンテナレジストリ(OCIR)への push
4. 更新した Docker イメージのタグを Kubernetes の Manifest ファイルに反映(Kustomize 使用)
5. Manifest ファイルの変更を検知し、変更があった場合は Manifest ファイルを取得する
6. Manifest の変更を Kubernetes クラスタへ反映する

まずはこれを実現するための、GitHub Actions の設定ファイルを以下のように書きます。

```.github/workflows/docker-image.yaml
name: Docker Image CI

on:
  push:
    branches: [main]

jobs:
  buildx:
    name: build
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Login to OCIR # ... 1
        uses: docker/login-action@v1
        with:
          registry: nrt.ocir.io
          username: ${{ secrets.OCIR_USERNAME }}
          password: ${{ secrets.OCIR_PASSWORD }}

      - name: Set up Docker Buildx # ... 2
        uses: docker/setup-buildx-action@v1
        id: buildx

      - name: Build Docker image and push to OCIR # ... 3
        run: |
          docker buildx build -t ${{ secrets.OCIR_BASE_PATH }}/greet:$GITHUB_SHA ./greet --platform linux/amd64,linux/arm64 --push

  deploy:
    name: deploy
    runs-on: ubuntu-latest
    needs: buildx

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Setup Kustomize # ... 4
        uses: imranismail/setup-kustomize@v1

      - name: Update Kubernetes resources # ... 5
        run: |
          cd deploy/kubernetes/overlays
          kustomize edit set image ${{ secrets.OCIR_BASE_PATH }}/greet=${{ secrets.OCIR_BASE_PATH }}/greet:$GITHUB_SHA
          cat kustomization.yaml

      - name: Commit files # ... 6
        run: |
          git config --local user.email "shukawam@gmail.com"
          git config --local user.name "shukawam"
          git commit -am "Bump docker tag"

      - name: Push changes # ... 7
        uses: ad-m/github-push-action@master
        with:
          github_token: ${{ secrets.PERSONAL_TOKEN }}
```

簡単に解説していきます。

#### 1. Login to OCIR

GitHub Actions から OCIR に対してイメージを push するためにログインしておきます。この時、OCIR にログインするために必要な資格情報等はリポジトリの `secrets` に定義しています。

![image06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/432c393c-1350-8cae-34b8-a2019013bc73.png)

#### 2. Set up Docker Buildx

[Docker のリポジトリ](https://github.com/docker)から公開されている[Buildx のセットアップ用の Action](https://github.com/docker/setup-buildx-action)を使っています。本例以外にも`with`パラメータで色々設定ができます。

| Name              | Type   | Description                                                                                                                                                                     |
| ----------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `version`         | String | [buildx](https://github.com/docker/buildx) version. (eg. `v0.3.0`, `latest`, `https://github.com/docker/buildx.git#master`)                                                     |
| `driver`          | String | Sets the [builder driver](https://github.com/docker/buildx/blob/master/docs/reference/buildx_create.md#driver) to be used (default `docker-container`)                          |
| `driver-opts`     | CSV    | List of additional [driver-specific options](https://github.com/docker/buildx/blob/master/docs/reference/buildx_create.md#driver-opt) (eg. `image=moby/buildkit:master`)        |
| `buildkitd-flags` | String | [Flags for buildkitd](https://github.com/moby/buildkit/blob/master/docs/buildkitd.toml.md) daemon (since [buildx v0.3.0](https://github.com/docker/buildx/releases/tag/v0.3.0)) |
| `install`         | Bool   | Sets up `docker build` command as an alias to `docker buildx` (default `false`)                                                                                                 |
| `use`             | Bool   | Switch to this builder instance (default `true`)                                                                                                                                |
| `endpoint`        | String | [Optional address for docker socket](https://github.com/docker/buildx/blob/master/docs/reference/buildx_create.md#description) or context from `docker context ls`              |
| `config`          | String | [BuildKit config file](https://github.com/docker/buildx/blob/master/docs/reference/buildx_create.md#config)                                                                     |

※[https://github.com/docker/setup-buildx-action](https://github.com/docker/setup-buildx-action)より引用

#### 3. Build Docker image and push

[Docker Buildx](https://matsuand.github.io/docs.docker.jp.onthefly/buildx/working-with-buildx/)を使ってマルチアーキテクチャ(AMD, ARM)対応するための Docker イメージを作成し OCIR に対して push しています。また、作成したイメージに付与するタグが衝突しないように簡易的にコミット SHA をイメージのタグに設定しています。

#### 4. Setup Kustomize

環境差分(と言っても今回は 1 環境しかないですが...)を管理するために [Kustomize](https://kustomize.io/) を使っています。使い慣れているので、Kustomize を使っていますが、Helm とかでも良いと思います。

#### 5. Update Kubernetes resource

"Build Docker image and push" で作成した Docker イメージのタグで Manifest の内容を上書きしています。

#### 6. Commit files & 7. Push changes

GitHub Actions 中で行われた変更を Manifest が保管されているリポジトリ(今回はソースコードが管理されているリポジトリと一緒)に反映しています。

### (おまけ)ちゃんとやるなら

今回は簡易的に実施するため、ソースコード用のリポジトリと Manifest ファイル用のリポジトリを同一にしていますが、本来であれば以下のような構成が望ましいと思います。

![image05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/96e1ab12-7611-a9e6-e3fc-8aa5ed6b1f0c.png)

また、イメージスキャンやポリシーチェック等も今回は実施していないので、その辺りまできちんと実施したい場合は [Oracle Cloud Hangout Cafe - CI/CD 最新事情](https://speakerdeck.com/oracle4engineer/oracle-cloud-hangout-cafe-cicdzui-xin-shi-qing) が非常に参考になると思います。

## OKE で ARM コンテナを動かす

まずは、OKE の Node Pool を追加します。新しくクラスタを作成する方は、検証目的ならクイック作成がおすすめです。

![image08.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/c6618787-a279-d1dd-44eb-ccc5e644bc85.png)

シェイプを `VM.Standard.A1.Flex` にする以外は自由に入力してください。

![image09.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/9f628dbf-5490-11f1-57b0-37abc1bfc0e8.png)

これで、1 つの OKE クラスタに

- AMD ベースの Node Pool
- ARM ベースの Node Pool

が混在するような構成を作ることができました。さっそくこのクラスタに対して先ほどの CI/CD パイプライン構築の続きをしていきます。

### Argo CD on OKE

Argo CD 自体の環境は、[ArgoCD - Getting Started](https://argoproj.github.io/argo-cd/getting_started/) に沿って実施してください。と、言いたいところですが、現在 Getting Started から公開されている手順とは若干異なるので、少し言及しておきます。(関連 Issue は [https://github.com/argoproj/argo-cd/issues/6787](https://github.com/argoproj/argo-cd/issues/6787) です)

```bash
# create namespace "argocd"
kubectl create namespace argocd
# apply manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

次に、Getting Started では初期パスワードが `argocd-initial-admin-secret` という secret に保存されているのでそれを参照してログインを行うのですが、私の環境(Argo CD v2.0.4)では、

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
Error from server (NotFound): secrets "argocd-initial-admin-secret" not found
```

と、そんな secret ないよとエラーが出力されてしまったので、admin ユーザーのパスワードを変更することで対応しました。([FAQ](https://argoproj.github.io/argo-cd/faq/#i-forgot-the-admin-password-how-do-i-reset-it) より)

```bash
kubectl -n argocd patch secret argocd-secret \
> -p '{"stringData": {"admin.password": "$2a$10$qo3Q/Zt7UaYZgcapXCZ8jOdcnU1Z/PXe8Y4EG.JHiLZUCoBVQ9sFq", "admin.passwordMtime": "'$(date +%FT%T%Z)'"}}'
secret/argocd-secret patched
```

なお、パスワードは bcrypt でハッシュ化する必要があるので、[この辺り](https://www.browserling.com/tools/bcrypt)を使うと楽にできます。

ここからアプリケーションの作成をします。まずは、リポジトリの定義をします。(作成したリポジトリの URL を指定します)

![image10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/b4cfa438-cc88-b6f4-3213-f10a1003b507.png)

次にアプリケーションの定義です。

![image11.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/3ddac201-c930-d854-aa7d-059046d88fce.png)

![image12.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/5d63d5bf-b9a2-7785-be81-85f5c9fc6f5c.png)

![image13.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/c5f72220-f808-f60b-14b7-a79e2a5b83c7.png)

尚、今回は以下のような Manifest ファイルを用意しました。

AMD 用:

```greet-amd.yaml
apiVersion: v1
kind: Service
metadata:
  name: greet-amd
  annotations:
    service.beta.kubernetes.io/oci-load-balancer-shape: 10Mbps
spec:
  selector:
    app: greet-amd
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: greet-amd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: greet-amd
  template:
    metadata:
      labels:
        app: greet-amd
    spec:
      containers:
        - name: greet-amd
          image: nrt.ocir.io/orasejapan/shukawam/greet
          ports:
            - name: api
              containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health/ready
              port: api
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health/live
              port: api
            initialDelaySeconds: 15
            periodSeconds: 20
          imagePullPolicy: Always
      imagePullSecrets:
        - name: greet-secret
      nodeSelector:
        beta.kubernetes.io/arch: amd64
```

ARM 用:

```greet-arm.yaml
apiVersion: v1
kind: Service
metadata:
  name: greet-arm
  annotations:
    service.beta.kubernetes.io/oci-load-balancer-shape: 10Mbps
spec:
  selector:
    app: greet-arm
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: greet-arm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: greet-arm
  template:
    metadata:
      labels:
        app: greet-arm
    spec:
      containers:
        - name: greet-arm
          image: nrt.ocir.io/orasejapan/shukawam/greet
          ports:
            - name: api
              containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health/ready
              port: api
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health/live
              port: api
            initialDelaySeconds: 15
            periodSeconds: 20
          imagePullPolicy: Always
      imagePullSecrets:
        - name: greet-secret
      nodeSelector:
        beta.kubernetes.io/arch: arm64
```

`nodeSelector` で CPU アーキテクチャを選択しているのがポイントです。(今回の Docker イメージは ARM/AMD に両方対応しているので、`nodeSelector` で指定しなくても良しなに分散配置してくれるはずです。多分...)
tag の情報などは、[Kustomize](https://kustomize.io/) を使って各環境用に書き換えています。その辺りは、[リポジトリ](https://github.com/shukawam/oke-ocir-multi-architecture)をご参照ください。

### OKE on ARM (and AMD)

実際にデプロイされた Pod を見てみましょう。

```bash
kubectl get pods,service -n multi-arch
NAME                             READY   STATUS    RESTARTS   AGE
pod/greet-amd-5749f798fd-9vfmt   1/1     Running   0          25m
pod/greet-arm-667d4b5946-pq4pk   1/1     Running   0          25m

NAME                TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
service/greet-amd   LoadBalancer   10.96.222.236   129.159.116.65   8080:31041/TCP   60m
service/greet-arm   LoadBalancer   10.96.243.175   152.70.206.253   8080:30013/TCP   60m
```

無事、Pod が二つデプロイされています。それぞれの詳細を見てみましょう。まずは、作成した OKE のクラスターに含まれる Node の情報を見てみます。

```bash
kubectl get nodes
NAME          STATUS   ROLES   AGE   VERSION
10.0.10.12    Ready    node    13h   v1.20.8
10.0.10.165   Ready    node    15h   v1.20.8
```

さて、この Node ですが、OCI コンソールから確認してみると確かに AMD と ARM のインスタンスから構成されている事が分かります。(10.0.10.12: AMD, 10.0.10.165: ARM)

![image14.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/8aa524f2-114e-cc1d-eb3c-fdda335a2883.png)

デプロイされた Pod の詳細をそれぞれ見てみます。

```bash
kubectl describe pod greet-amd-5749f798fd-9vfmt -n multi-arch
Name:         greet-amd-5749f798fd-9vfmt
Namespace:    multi-arch
Priority:     0
Node:         10.0.10.12/10.0.10.12
Start Time:   Sun, 25 Jul 2021 06:26:39 +0000
Labels:       app=greet-amd
              pod-template-hash=5749f798fd
Annotations:  <none>
Status:       Running
IP:           10.244.1.17
IPs:
  IP:           10.244.1.17
# ... 省略
```

`Node: 10.0.10.12/10.0.10.12` から確かに、AMD の Node に配置されている事が分かります。次に、もう片方の Pod の詳細を見てみます。

```bash
kubectl describe pod greet-arm-667d4b5946-pq4pk -n multi-arch
Name:         greet-arm-667d4b5946-pq4pk
Namespace:    multi-arch
Priority:     0
Node:         10.0.10.165/10.0.10.165
Start Time:   Sun, 25 Jul 2021 06:26:39 +0000
Labels:       app=greet-arm
              pod-template-hash=667d4b5946
Annotations:  <none>
Status:       Running
IP:           10.244.0.26
IPs:
  IP:           10.244.0.26
# ... 省略
```

こちらも `Node: 10.0.10.165/10.0.10.165` から確かに、ARM の Node に配置されている事が分かります。

最後に、それぞれ Load Balancer を通して公開されているエンドポイントを叩いてみます。

AMD:

```bash
curl http://129.159.116.65:8080/greet
{"message":"Hello World!"}
```

ARM:

```bash
curl http://152.70.206.253:8080/greet
{"message":"Hello World!"}
```

# 終わりに

想像しているよりも簡単にヘテロジニアスな環境を作ることが出来ました。ARM インスタンスはかなり安い料金設定(コア時間当たり 1.2 円)なのでうまく活用してコストダウンを図りたいですね！

また、今回作成した一連のファイルは、[リポジトリ](https://github.com/shukawam/oke-multi-architecture)に格納しているので、良ければ参考にしてください。

# 参考

- [Oracle Container Engine for Kubernetes（OKE）のArmインスタンスのサポート](https://blogs.oracle.com/oracle4engineer/oke-arm-support)
- [GitHub Actions - Setup buildx action](https://github.com/docker/setup-buildx-action)
- [Oracle Cloud Hangout Cafe - CI/CD 最新事情](https://speakerdeck.com/oracle4engineer/oracle-cloud-hangout-cafe-cicdzui-xin-shi-qing)
- [ArgoCD - Getting Started](https://argoproj.github.io/argo-cd/getting_started/)
