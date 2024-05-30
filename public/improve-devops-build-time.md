---
title: OCI DevOps を使った Node.js アプリケーションのビルド時間を短縮する
tags:
  - oci
  - oraclecloud
  - CICD
private: false
updated_at: '2023-04-18T16:46:38+09:00'
id: 6e413ad6c3ee19d51801
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

以前、こんな記事を書きました。

https://qiita.com/shukawam/items/62a002a42a73a4e5ca7e

npm 系での扱い方も需要がありそうなので再度取り上げたいと思います。

# ポイント

ポイントは、生成されたキャッシュを Object Storage に保存し、2 回目以降のビルド時にそれを活用することです。これを実現するための build_spec.yaml を見てみましょう。

```build_spec.yaml
version: 0.1
component: build
timeoutInSeconds: 10000
shell: bash
env:
  variables:
    BUILD_CACHE_OS_BUCKET_NAME: build-cache
    BUILD_CACHE_OS_FILE_NAME: node_modules.tar
​
steps:
  - type: Command
    name: "Build Cache Restore"
    command: |
      echo ${OCI_PRIMARY_SOURCE_DIR}
      cd ${OCI_PRIMARY_SOURCE_DIR}
      oci os object get \
        --bucket-name ${BUILD_CACHE_OS_BUCKET_NAME} \
        --file ${BUILD_CACHE_OS_FILE_NAME} \
        --name ${BUILD_CACHE_OS_FILE_NAME} \
      && tar xf ${BUILD_CACHE_OS_FILE_NAME}
      echo "Done..."
​
  - type: Command
    name: "Install dependencies"
    command: |
      npm install
​
  - type: Command
    name: "Build application"
    command: |
      npm run build
    onFailure:
      - type: Command
        command: |
          echo "Failure successfully handled"
        timeoutInSeconds: 60
​
  - type: Command
    name: "Build Cache Upload"
    command: |
      cd ${OCI_PRIMARY_SOURCE_DIR}
      tar cf ${BUILD_CACHE_OS_FILE_NAME} node_modules
      oci os object put \
        --bucket-name ${BUILD_CACHE_OS_BUCKET_NAME} \
        --file ${BUILD_CACHE_OS_FILE_NAME} \
        --force
```

順番に解説していきます。

```yaml
env:
  variables:
    BUILD_CACHE_OS_BUCKET_NAME: build-cache
    BUILD_CACHE_OS_FILE_NAME: node_modules.tar
```

ここでは、パイプライン中に使用する変数を宣言しています。

- BUILD_CACHE_OS_BUCKET_NAME: キャッシュを保存しておくバケット名
- BUILD_CACHE_OS_FILE_NAME: バケットに配置するファイル名

```yaml
- type: Command
  name: "Build Cache Restore"
  command: |
    echo ${OCI_PRIMARY_SOURCE_DIR}
    cd ${OCI_PRIMARY_SOURCE_DIR}
    oci os object get \
      --bucket-name ${BUILD_CACHE_OS_BUCKET_NAME} \
      --file ${BUILD_CACHE_OS_FILE_NAME} \
      --name ${BUILD_CACHE_OS_FILE_NAME} \
    && tar xf ${BUILD_CACHE_OS_FILE_NAME}
    echo "Done..."
```

ここでは、Object Storage に格納した `node_modules.tar` をダウンロードし、展開しています。該当のバケットにファイルが存在しない場合も後続のステージは続行されます。

```yaml
- type: Command
  name: "Install dependencies"
  command: |
    npm install
```

依存関係のインストールが行われます。今回の CI プロセスのうち、Object Storage を利用したキャッシュによって時間の短縮が期待できるのはこの部分になります。

```yaml
- type: Command
  name: "Build application"
  command: |
    npm run build
  onFailure:
    - type: Command
      command: |
        echo "Failure successfully handled"
      timeoutInSeconds: 60
```

アプリケーションのビルドが行われます。

```yaml
- type: Command
  name: "Build Cache Upload"
  command: |
    cd ${OCI_PRIMARY_SOURCE_DIR}
    tar cf ${BUILD_CACHE_OS_FILE_NAME} node_modules
    oci os object put \
      --bucket-name ${BUILD_CACHE_OS_BUCKET_NAME} \
      --file ${BUILD_CACHE_OS_FILE_NAME} \
      --force
```

`node_modules/*` を tar に固めて Object Storage にアップロードしています。同一ファイル名のファイルが存在する場合、上書きさせるため `--force` オプションを追加していますが、この辺りはご自由にどうぞ。

最後に結果を確認してみましょう。今回は、`npx create-next-app@latest` で作ったシンプルな Next.js のアプリケーションで確認しています。

1 回目: 3 分 0 秒

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/11453c30-ac69-88b1-9633-46cd5c7325d2.png)

2 回目: 2 分 29 秒

![image02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/e580aafc-7d77-6115-1a8c-be435de142a6.png)

確かに、依存ライブラリのインストール時間が短縮されたことが確認できました。（16 秒 → 4 秒）

また、1 回目のビルドが完了すると、Object Storage に node_modules.tar というファイルが生成されていることが確認できます。

![image03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/f75ff416-a815-b917-df92-ed68fede3f4e.png)

# 終わりに

今回は、シンプルなアプリケーションで依存ライブラリが少なかったため、この程度の差となりましたが、依存関係が増えた場合はさらに CI プロセスの時間短縮が期待できるのではないでしょうか。
