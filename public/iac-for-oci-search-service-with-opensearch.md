---
title: OCI Search Service with OpenSearch を使うための最低限の環境を IaC 化した
tags:
  - oracle
  - IaC
  - Terraform
  - oci
  - OpenSearch
private: false
updated_at: '2022-12-02T07:02:28+09:00'
id: 4e6866dbf3d65d6657ec
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

こちらの記事は、[Oracle Cloud Infrastructure Advent Calendar 2022](https://qiita.com/advent-calendar/2022/oci) の Day 2 の記事として書かれています！

コスト削減の意味で OpenSearch のクラスターが必要になった都度作り直していたのですが、毎度毎度めんどくさいなと思い、OCI Search Service with OpenSearch を扱うための OCI リソース一式を IaC 化してみました。

## 作成する環境

Resource Manager(OCI が提供する Terraform のマネージド・サービス)から `terraform apply` することで以下のような環境を作ります。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/248fe772-b0d4-3850-cbbf-553749724e8c.png)

デフォルトでは、各ノード（Master/Dashboard/Data）が 1 台ずつ最小構成でプロビジョニングされます。具体的には以下の通りです。

- Master Node
  - OCPU: 1
  - Memory: 20GB
- Dashboard Node
  - OCPU: 1
  - Memory: 8GB
- Data Node
  - OCPU: 1
  - Memory: 20GB
  - Storage: 50GB

## 作成手順

[https://github.com/shukawam/resource-manager/tree/main/examples/opensearch](https://github.com/shukawam/resource-manager/tree/main/examples/opensearch) にアクセスします。
README に `Deploy to Oracle Cloud` というボタンがあるので、こちらをクリックします。

![image02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/b34487fd-c9da-ae6e-77a4-c33c8a640884.png)

クリックすると、現在使用中のリージョンで Resource Manager のスタック作成画面に遷移します。

![image03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/cc78bfbd-5164-57c2-4a2b-3b7e22f9fe0f.png)

変数の構成で入力必須な項目を入力します。

- compartment_id: 先の構成がプロビジョニングされる Compartment OCID を指定
- opensearch_cluster_display_name: クラスターの表示名を指定
- private_key_location: 踏み台に SSH 接続するための自身の秘密鍵のパスを指定
- ssh_public_key: 踏み台に SSH 接続するための公開鍵の本文を指定

その他は、任意項目のため好きに調整してみてください。

次に、作成されたスタックの詳細画面から適用（`terraform apply`に相当します）をクリックします。  
※ちゃんとやる方は、計画（`terraform plan`に相当）してから作成された計画を元に適用してください）

![image04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/fdb65e09-625b-3d23-33c6-d2665081db7a.png)

15~20 分ほど経過するとジョブが完了します。

![image05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/5f8b02fe-32fe-ca43-bbfd-6560a6b0cfc3.png)

完了したジョブの詳細画面からログを参照します。ログの末尾に出力変数として、手元の端末からポートフォワードするためのスクリプトを出力しているので、これを手元で実行します。

![image06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/8bec9b9b-dbdf-cb38-e253-b4554f757cc4.png)

実行例

```bash
ssh -C -v -t -L 127.0.0.1:5601:10.0.0.10:5601 -L 127.0.0.1:9200:10.0.0.186:9200 opc@129.146.21.98 -i <private-key-path>
# ... omit

[opc@opensearch-bastion ~]$
```

手元のブラウザで `https://localhost:5601` にアクセスし、以下の画面が参照できれば無事完了です。

![image07.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/9b8f822c-5c25-b9e5-b975-49bc5af8177b.png)


# おわりに

何か動かない等の不備があれば、[GitHub - Issues](https://github.com/shukawam/resource-manager/issues) までよろしくお願いいたします。
