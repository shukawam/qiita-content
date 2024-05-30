---
title: OCI DevOps で Oracle Functions の CI/CD - パイプライン作成編
tags:
  - oci
  - oraclecloud
  - CICD
private: false
updated_at: '2022-01-19T17:15:06+09:00'
id: d345844fbd1e0a9c4712
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

こちらの記事は、[Oracle Cloud Infrastructure Advent Calendar 2021](https://qiita.com/advent-calendar/2021/oci) その 2 の Day 5 の記事として書かれています。
今回は、10 月後半に全機能が揃った [OCI DevOps](https://docs.oracle.com/ja-jp/iaas/Content/devops/using/home.htm) に関する記事です。OKE の方はきっと誰かが書いてくれることを願って私の方では Oracle Functions に特化した内容で書きたいと思います。内容が結構ボリューミーになりそうなので全 2 回に分けたいと思います。

**2022/01/19 追記**: [Oracle Cloud Infrastructure(OCI) DevOpsことはじめ](https://oracle-japan.github.io/ocitutorials/cloud-native/devops-for-commons/)
OCI Tutorials に DevOps & OKE のハンズオンが追加されたので、OKE で同様の事を実施したい場合はこちらをご参照ください。

1. [事前準備編](https://qiita.com/shukawam/private/61232a6fd1071b861c83)
2. **ビルド、デプロイメント・パイプライン作成編** ← 本記事はこれ

今回は、ビルド、デプロイメント・パイプライン作成編です。

# 今回作る環境

全 2 回を通してこのような環境を作ってみます。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/d5587bf5-f748-3439-d9e0-ff6b958aa3f8.png)

ソースコード本体は、コミュニケーション機能（Issue, Wiki, etc.）が充実している GitHub, GitLab で管理し、バックアップを取るためにソースコードを Code Repository へミラーリングする実際の開発現場でよく取られそうな構成です。

# 実際に作ってみる

## 前提

- [OCI DevOps で Oracle Functions の CI/CD - 事前準備編](https://qiita.com/shukawam/private/61232a6fd1071b861c83)が完了していること

## ビルド・パイプライン

GitHub からのミラーリングの設定 ~ OCIR[^1] に Oracle Functions の Docker Image を push する所までを作っていきます。図で表すと以下の赤線の箇所を作成していきます。

![image17.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/75e16358-df02-d981-5cb9-198a99defe5b.png)

[^1]: Oracle Container Infrastructure Refistry

### 外部接続の定義

まずは、GitHub → Code Repository に対するミラーリングの設定を行うための接続定義を行います。OCI コンソール左上のハンバーガーメニューから**開発者サービス** > **プロジェクト**と選択します。

![image18.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/822117e0-0ff4-c361-f090-985e11da0b97.png)

先ほど、作成した DevOps のプロジェクトの詳細画面から**外部接続**を選択します。

![image19.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/350022e9-daac-2da7-928f-28f24e93cf32.png)

**外部接続の作成**を押し、以下のように入力して外部接続を新しく定義します。

- 名前: github-connection
- タイプ: GitHub
- ボールト・シークレット
  - ボールト: 作成済みの Vault(GitHub の PAT が格納されている Vault を選択する)
  - シークレット: GitHub の PAT

![image20.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/c9fddffc-301e-57a1-cf9d-f653e73c17fd.png)

これで、ミラーリングのための外部接続設定は完了です。

### リポジトリのミラーリング設定

ミラーリングの設定を行います。作成済みの DevOps プロジェクトの詳細画面から**コード・リポジトリ**を選択します。

![image21.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/01de5dbd-2d79-e48f-96b9-89757876ae12.png)

**リポジトリのミラー化**を選択します。

![image22.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/e423d7eb-af31-185e-0582-f4363cbd0bcb.png)

以下のように入力して、ミラー先のリポジトリを定義します。

- 接続: github-connection
- リポジトリ: fn-examples
- ミラーリング・スケジュール
  - スケジュール・タイプ: カスタム
  - ミラーリング間隔: 15 分(最短 1 分から設定可能なので、お好きな値に設定してください)
- 名前: github_fn-examples

これで、ミラー先のリポジトリの作成が完了です。

### build_spec.yaml の作成

いよいよ、ビルドを行うための設定ファイルを書いていきます。build_spec.yaml の仕様に関しては [OCI Document - ビルド指定](https://docs.oracle.com/ja-jp/iaas/Content/devops/using/build_specs.htm)をご参照ください。今回は、以下のような設定ファイルを `fn-examples/fn-hello/`に配置します。

```build_spec.yaml
# https://docs.oracle.com/ja-jp/iaas/Content/devops/using/build_specs.htm
# build runner config
version: 0.1
component: build
timeoutInSeconds: 10000
shell: bash
env: # ... 1
  variables:
    function_name: fn-hello
    function_version: 0.0.2
  exportedVariables:
    - tag

# steps
steps:
  - type: Command
    name: "Docker image build"
    timeoutInSeconds: 4000
    command: |
      cd fn-hello
      fn build
      docker tag ${function_name}:${function_version} fn-hello-image
      tag=${function_version}
    onFailure:
      - type: Command
        command: |
          echo "Failure successfully handled"
        timeoutInSeconds: 60

outputArtifacts:
  - name: fn-hello-image
    type: DOCKER_IMAGE
    location: fn-hello-image
```

いくつか、ポイントがあるので簡単に補足します。

1. パイプライン中に使用する変数を定義しています。
   - variables(パイプライン中に使用する変数)
     - function_name: 作成した関数名（fn-hello）
     - function_version: func.yaml に記載のある関数のバージョン
   - exportedVariables(後続のステージに渡す変数を定義)
     - tag: 実際に OCIR に push されるときの Docker Image のバージョン
2. Docker Image のビルドを Fn CLI のビルドオプション（`fn build`）を用いて実施しています。また、作成される Docker Image は今回の場合だと `fn-hello:0.0.2` という名前が付けられているので、それを `docker tag ...` で `fn-hello-image` に変更しています。次に、実際に push する際のバージョンを `tag=${function_version}`で設定し、アーティファクトの配信ステージで活用します。（単純に Commit Hash(`${OCI_TRIGGER_COMMIT_HASH}`)を使うのでも良いと思います。）
3. 次のステージ（アーティファクトの配信）で使うためのアーティファクトを指定します。

作成した build_spec.yaml は GitHub に push しておきます。

```bash
# ./fn-examples
git add fn-hello/build_spec.yaml
```

```bash
git commit -m "Added build_spec.yaml"
```

```bash
git push origin main
```

### ビルド・パイプラインの作成

前工程で作成した build_spec.yaml を実際に適用するビルド・パイプラインを作成していきます。作成済みの DevOps プロジェクトの詳細画面から**ビルド・パイプライン**を選択します。

![image23.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/902f8780-963a-58bd-dfbb-19c5591a66b4.png)

**ビルド・パイプラインの作成**をクリックし、以下のように入力しビルド・パイプラインを新規に作成します。

- 名前: fn-hello-build-pipeline

![image24.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/ec9198d2-a086-44cb-d13f-e94861b4273a.png)

作成したビルド・パイプラインの詳細画面で**ステージの追加**を押し、新規にステージを作成します。

![image25.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/be9d26a9-2714-b555-3ed4-a4ec8fc765e1.png)

ステージ・タイプの選択で**マネージド・ビルド**を選択し、**次**を押します。

![image26.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/6bbc6ef6-54cf-086d-ed2a-8ec253094ff1.png)

以下のように入力し、ビルド・ステージを追加します。

- ステージ名: build_stage
- ビルド指定ファイル・パス: fn-hello/build_spec.yaml
- プライマリ・コード・リポジトリ
  - ソース: 接続タイプ: github_fn-examples
  - ブランチの選択: main
  - ソース名の作成: fn-hello

![image27.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/eea0d1ba-6554-0317-a441-29063e084045.png)

次に、build_stage で生成し outputArtifact として出力した Oracle Functions の Docker Image を OCIR へプッシュするステージを作成します。build_stage 下の"+"アイコンをクリックし、ステージの追加を選択します。

![image30.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/599bea43-4d0a-033d-1f41-b5fe52f5619c.png)

ステージ・タイプとして**アーティファクトの配信**を選択します。

![image31.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/27c2cc57-8461-ff1c-f9ac-c8d040a0a94e.png)

以下のように入力します。

- ステージ名: image_push
- アーティファクトの作成を選択
  - 名前: fn-hello-image
  - タイプ: コンテナ・イメージ・リポジトリ
  - コンテナ・レジストリへの完全修飾パス: \<region-key\>.ocir.io/\<namespace\>/devops/fn-hello:${tag}
        - \<region-key\>の詳細は、[Oracle Cloud Infrastructure Registry へのログイン](https://docs.oracle.com/ja-jp/iaas/Content/Functions/Tasks/functionslogintoocir.htm)をご参照ください
  - このアーティファクトで使用するパラメータの置き換え: はい、プレースホルダーを置き換えます
- ビルド構成/結果アーティファクト名: build_spec.yaml で outputArtifact として指定した名前（fn-hello-image）

![image32.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/ddf0f976-7caa-f9a9-9eb4-bc93163d93d9.png)

これで、ビルドパイプラインが起動された際に Oralce Functions の Docker Image の作成と OCIR へプッシュを行うビルド・パイプラインが作成できました。

### トリガーの設定

作成したビルド・パイプラインを起動するためのトリガーを作成します。作成済みの DevOps プロジェクトの詳細画面から**トリガー**を選択します。

![image28.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/a78b4ab8-075d-7409-b254-e200b48e7937.png)


**トリガーの作成**を押し、以下のように入力します。

- 名前: fn-hello-trigger
- ソース接続: OCI コード・リポジトリ
- コード・リポジトリの選択: github_fn-examples
- アクションの追加
  - ビルド・パイプラインの選択: fn-hello-build-pipeline
  - イベント: プッシュにチェックを入れる

![image29.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/95d84289-ee50-334d-3205-93c4b31874e2.png)

これで、GitHub へソースコードの変更を push し OCI の Code Repository と同期がとられたタイミングで作成したビルド・パイプラインが動作するところまで作成することが出来ました。

![image17.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/83848226-46df-bfd9-b85f-5b1457f11de8.png)

## デプロイメント・パイプライン

いよいよ、ビルド・パイプラインで作成した Docker Image を Oracle Functions へデプロイするパイプラインを作成していきます。図で表すと以下の赤線の箇所を作成していきます。

![image45.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/f19c4653-4475-2e8d-da93-d32668f5b28c.png)

### 環境の作成

まずは、デプロイ先の環境を定義します。作成済みの DevOps プロジェクトの詳細画面から**環境**を選択します。

![image33.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/b0725214-2887-2908-a5a8-27bca2d911a1.png)

**環境の作成**を押し、以下のように入力します。

- 環境タイプ: ファンクション
- 名前: fn-hello-env

![image34.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/1620dd72-d375-0757-f826-a8233c61a46c.png)

環境詳細で以下のように入力します。

- リージョン: 自分がリソースを配置するリージョンを選択
- コンパートメント: 自分がリソースを配置するコンパートメントを選択
- アプリケーション: devops-app
- ファンクション: fn-hello

![image35.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/f4822be7-b629-d435-dc33-fa84d08ad08c.png)

### デプロイメント・パイプラインの作成

定義した環境にデプロイするためのパイプラインを作成します。作成済みの DevOps プロジェクトの詳細画面から**デプロイメント・パイプライン**を選択します。

![image36.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/880f0fa9-d5cf-07f8-9a0f-337c6cd0bea4.png)

**パイプラインの作成**を押し、以下のように入力します。

- パイプライン名: fn-hello-deploy-pipeline

![image37.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/ed2153c0-8c7c-d71a-64b0-bfac21e86040.png)

**ステージの追加**を押します。

![image38.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/05f13709-931f-fcec-4700-bafde8c8a1ae.png)

タイプとして、ファンクション用のステージを選択します。

![image39.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/6f280415-95bd-3d68-49ca-0143e8209929.png)

以下のように入力します。

- ステージ名: deploy_stage
- 環境: fn-hello-env
- アーティファクトの選択から、fn-hello-image を選択します

![image40.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/c697e594-340c-c1f5-6607-ee9aaf3b68ad.png)

これで、デプロイメント・パイプラインの作成は完了です。

### ビルド・パイプラインの連携設定

最後に、作成したビルド・パイプラインとデプロイメント・パイプラインを連携させます。作成済みの DevOps プロジェクトの詳細画面から**ビルド・パイプライン**を選択します。

![image23.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/f5d35614-3517-c83e-900c-edf2128db53f.png)

先ほどの手順で作成した **fn-hello-build-pipeline** を選択します。image_push の下の"+"アイコンを押します。

![image41.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/f8576a12-cedb-0c77-d097-799665924fc5.png)

ステージの選択で**デプロイメントのトリガー**を選択します。

![image42.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/ed7fabb2-9fd4-1835-5121-3152809c4936.png)

以下のように入力します。

- ステージ名: deploy_stage
- デプロイメント・パイプラインの選択を押し、作成済みの fn-hello-deploy-pipeline を選択します。

![image43.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/98cf44e3-cf17-3097-d05e-f6460ae5af26.png)

最終的に以下のような状態になっていればパイプライン作成は完了です！

![image44.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/05f85f20-5d43-6845-9d06-326f24977322.png)

## 実際に動かしてみる

今回は、手動で実行します。ビルド・パイプラインから**手動実行の開始**をクリックします。

しばらく時間が経過すると、パイプラインの実行が成功したことが確認できます。

![image46.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/5d486927-a0dd-7efe-f8ac-3f6bafa53418.png)

また、OCIR にて build_spec.yaml で設定した`${tag}`のイメージが生成されていることからも確認できます。

![image47.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/f6c30958-75fd-62d1-5db1-4372381a2285.png)

以降は、ソースコード等に対する変更が行われ、Code Repository にその変更が反映（ミラー）されたことをトリガーとし、パイプラインが実行されます。

# 終わりに

OCI Native な CI/CD サービスが全て揃ったので、さらに開発体験が良くなりますね！是非試してみてください！
また、今回使用したソースコード等は [https://github.com/shukawam/fn-examples](https://github.com/shukawam/fn-examples) に格納しています。

# 参考

- [DevOps ドキュメント](https://docs.oracle.com/ja-jp/iaas/Content/devops/using/home.htm)
