---
title: OCI DevOps を使った Java アプリケーションのビルドを高速化する
tags:
  - Java
  - oci
  - oraclecloud
  - CICD
  - BuildKit
private: false
updated_at: '2022-06-16T09:49:56+09:00'
id: 62a002a42a73a4e5ca7e
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

OCI DevOps を使った Java アプリケーションのビルド（コンテナイメージのビルド）を高速化する方法を紹介します。

# ポイント

ポイントは、

- BuildKit(Docker Buildx) を使う
- 生成されたキャッシュを Object Storage に保存し、2 回目以降のビルド時にそれを活用する

です。それを実現するための build_spec.yaml と Dockerfile を見てみましょう。

```build_spec.yaml
version: 0.1
component: build
timeoutInSeconds: 10000
shell: bash
env:
  variables:
    BUILD_CACHE_OS_BUCKET_NAME: build-cache
    BUILD_CACHE_OS_FILE_NAME: k8s-helidon-app-cache.zip
  exportedVariables:
    - TAG

steps:
  - type: Command
    name: "Export variables"
    command: |
      TAG=`echo ${OCI_BUILD_RUN_ID} | rev | cut -c 1-7`
      echo "TAG:" ${TAG}
    onFailure:
      - type: Command
        commnd: |
          echo "Failure successfully handled"
        timeoutInSeconds: 60
  - type: Command
    name: "Docker BuildKit Setup"
    timeoutInSeconds: 140
    command: |
      wget https://github.com/docker/buildx/releases/download/v0.8.2/buildx-v0.8.2.linux-amd64 -O docker-buildx
      mkdir -p ~/.docker/cli-plugins
      mv docker-buildx ~/.docker/cli-plugins/
      chmod +x ~/.docker/cli-plugins/docker-buildx
      docker buildx install
  - type: Command
    name: "Build Cache Restore"
    timeoutInSeconds: 140
    command: |
      oci os object get --bucket-name ${BUILD_CACHE_OS_BUCKET_NAME} --file ${BUILD_CACHE_OS_FILE_NAME} --name ${BUILD_CACHE_OS_FILE_NAME} && unzip ${BUILD_CACHE_OS_FILE_NAME}
      echo "Done..."
  - type: Command
    name: "Build Docker Image"
    command: |
      export DOCKER_BUILDKIT=1
      export DOCKER_CLI_EXPERIMENTAL=enabled
      docker buildx create --use
      docker buildx build -t=k8s-helidon-app --cache-from=type=local,src=./k8s-helidon-app-cache --cache-to=type=local,dest=./k8s-helidon-app-cache --load ${OCI_PRIMARY_SOURCE_DIR}
    onFailure:
      - type: Command
        command: |
          echo "Failure successfully handled"
        timeoutInSeconds: 60
  - type: Command
    name: "Build Cache Upload"
    timeoutInSeconds: 300
    command: |
      rm ${BUILD_CACHE_OS_FILE_NAME} && zip -r ${BUILD_CACHE_OS_FILE_NAME} k8s-helidon-app-cache/*
      oci os object put --bucket-name build-cache --file ${BUILD_CACHE_OS_FILE_NAME} --force

outputArtifacts:
  - name: k8s-helidon-app-image
    type: DOCKER_IMAGE
    location: k8s-helidon-app
```

順番に解説していきます。

```yaml
env:
  variables:
    BUILD_CACHE_OS_BUCKET_NAME: build-cache
    BUILD_CACHE_OS_FILE_NAME: k8s-helidon-app-cache.zip
```

ここでは、パイプライン中に使用する変数を宣言しています。

- BUILD_CACHE_OS_BUCKET_NAME: キャッシュを保存しておくバケット名
- BUILD_CACHE_OS_FILE_NAME: バケットに配置するファイル名

```yaml
steps:
  - type: Command
    name: "Export variables"
    command: |
      TAG=`echo ${OCI_BUILD_RUN_ID} | rev | cut -c 1-7`
      echo "TAG:" ${TAG}
    onFailure:
      - type: Command
        commnd: |
          echo "Failure successfully handled"
        timeoutInSeconds: 60
```

ここでは、ビルド毎の ID の一部を利用して、タグとして定義しています。定義したタグは、アーティファクトの識別に使用します。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/3582a853-5b34-7e61-3b39-fd97c06abe59.png)

```yaml
- type: Command
  name: "Docker BuildKit Setup"
  timeoutInSeconds: 140
  command: |
    wget https://github.com/docker/buildx/releases/download/v0.8.2/buildx-v0.8.2.linux-amd64 -O docker-buildx
    mkdir -p ~/.docker/cli-plugins
    mv docker-buildx ~/.docker/cli-plugins/
    chmod +x ~/.docker/cli-plugins/docker-buildx
    docker buildx install
```

OCI DevOps のビルド環境は、そのままでは BuildKit を使用することができないのでセットアップを行っています。

```yaml
- type: Command
  name: "Build Cache Restore"
  timeoutInSeconds: 140
  command: |
    oci os object get --bucket-name ${BUILD_CACHE_OS_BUCKET_NAME} --file ${BUILD_CACHE_OS_FILE_NAME} --name ${BUILD_CACHE_OS_FILE_NAME} && unzip ${BUILD_CACHE_OS_FILE_NAME}
    echo "Done..."
```

Object Storage に保存されているキャッシュを OCI CLI を用いて取得します。実施するためには、事前にポリシーの設定（DevOps のリソースを含む動的グループが特定のバケットに対する操作を行える事）が必要です。ポリシーの一例はこんな感じ。

```bash
Allow dynamic-group <dynamic-group> to read buckets in compartment <compartment-name>

Allow dynamic-group <dynamic-group> to manage objects in compartment <compartment-name> where all {target.bucket.name='build-cache'}
```

```yaml
- type: Command
  name: "Build Docker Image"
  command: |
    export DOCKER_BUILDKIT=1
    export DOCKER_CLI_EXPERIMENTAL=enabled
    docker buildx create --use
    docker buildx build -t=k8s-helidon-app --cache-from=type=local,src=./k8s-helidon-app-cache --cache-to=type=local,dest=./k8s-helidon-app-cache --load ${OCI_PRIMARY_SOURCE_DIR}
```

Docker Buildx を使うための各種設定と前ステップで取得したキャッシュを用いて、コンテナイメージのビルドを行います。この時に使用した Dockerfile は以下のよう。

```Dockerfile
# 1st stage, build the app
FROM maven:3.6-jdk-11 as build

WORKDIR /helidon

# Create a first layer to cache the "Maven World" in the local repository.
# Incremental docker builds will always resume after that, unless you update
# the pom
ADD pom.xml .
RUN --mount=type=cache,target=/root/.m2 mvn package -Dmaven.test.skip -Declipselink.weave.skip

# Do the Maven build!
# Incremental docker builds will resume here when you change sources
ADD src src
RUN mvn package
# RUN mvn package -DskipTests
RUN echo "done!"

# 2nd stage, build the runtime image
FROM openjdk:11-jre-slim
WORKDIR /helidon

# Copy the binary built in the 1st stage
COPY --from=build /helidon/target/k8s-helidon-app.jar ./
COPY --from=build /helidon/target/libs ./libs

CMD ["java", "-jar", "k8s-helidon-app.jar"]

EXPOSE 8080
```

`RUN --mount=type=cache,target=/root/.m2` の部分で .m2 のキャッシュが効くようにしています。

```yaml
- type: Command
  name: "Build Cache Upload"
  timeoutInSeconds: 300
  command: |
    rm ${BUILD_CACHE_OS_FILE_NAME} && zip -r ${BUILD_CACHE_OS_FILE_NAME} k8s-helidon-app-cache/*
    oci os object put --bucket-name build-cache --file ${BUILD_CACHE_OS_FILE_NAME} --force
```

最後にビルド時に生成されたキャッシュを再度 Object Storage へ OCI CLI を用いて格納しています。

アプリケーションの規模などにもよるかもしれませんが、これで大体ビルドに掛かっていた時間が半分以下となっています。是非、使っている方は試してみてください！

1 回目: 12 分 55 秒

![image02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/eb50be4c-1f19-61d0-e791-dbdda1bb7ed9.png)

2 回目: 3 分 44 秒

![image03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/aa4b0bf8-0002-5b65-462f-c0b46a943ca3.png)

# 終わりに

細かいところは、[リポジトリ](https://github.com/shukawam/k8s-helidon-app)を参照してください。
