---
title: Pulumi を使って Kubernetes 上にアプリケーションを構築してみた
tags:
  - TypeScript
  - IaC
  - kubernetes
  - Pulumi
  - OKE
private: false
updated_at: '2023-05-17T18:39:43+09:00'
id: 02401ca06e434c01a948
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

最近、IaC ツールとして [Pulumi](https://www.pulumi.com/) を使い始めてみました。類似のツールに [Terraform](https://www.terraform.io/) がありますが、GPL 故の表現力の高さであったり、初期の学習コストの低さや、使用言語のエコシステムが活用できる点で気に入っています。Provider の数としては正直まだまだ Terraform には劣りますが、主要な所は大体網羅されているように思います。

そこで、今回は Pulumi, [Kubernetes Provider](https://github.com/pulumi/pulumi-kubernetes) を使って簡単なアプリケーションを K8s 上（OKE を使用）に構築してみます。

# 全体像

以下の構成を作ってみます。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/cbb31b1c-8bda-e697-7982-2726e83b390e.png)

Java 製のアプリケーションを Ingress 経由で公開するシンプルな例です。別 Namespace(dev/prod)に対して同じアプリケーションをデプロイしますが、環境変数によってアプリケーションが返却するメッセージを変更しています。

# 手順

まずは、プロジェクトを作ります。

```bash
pulumi new kubernetes-typescript \
  --name k8s-helidon-app \
  --stack dev \
  --description "A TypeScript Pulumi program for simple kubernetes application."
```

実行すると、こんなファイル群が作成されます。

```bash
.
├── Pulumi.yaml
├── index.ts
├── node_modules
├── package-lock.json
├── package.json
└── tsconfig.json
```

生成された `index.ts` を確認してみると以下のようになっています。

```index.ts
import * as k8s from "@pulumi/kubernetes";
import * as kx from "@pulumi/kubernetesx";

const appLabels = { app: "nginx" };
const deployment = new k8s.apps.v1.Deployment("nginx", {
  spec: {
    selector: { matchLabels: appLabels },
    replicas: 1,
    template: {
      metadata: { labels: appLabels },
      spec: { containers: [{ name: "nginx", image: "nginx" }] },
    },
  },
});
export const name = deployment.metadata.name;
```

Kubernetes の Manifest ファイルを書いたことがある方でしたら特に迷うことなく自分のデプロイしたいリソースを表現できるかと思います。

ということで、先の構成を実現するべく以下のように実装してみました。以降で簡単に解説します。

```index.ts
import * as pulumi from "@pulumi/pulumi";
import * as pulumi from "@pulumi/pulumi";
import * as k8s from "@pulumi/kubernetes";
import * as kx from "@pulumi/kubernetesx";

interface Data {
  namespace: string;
  deployment: {
    replicas: number;
  };
  ingress: {
    host: string;
    tls: {
      hosts: string[];
      secretName: string;
    };
  };
  cowweb: {
    message: string;
  };
}

let config = new pulumi.Config();
const data = config.requireObject<Data>("data");

const appName = "k8s-helidon-app";
const appLabels = { app: appName };

function defaultProbeGenerator(path: string) {
  return {
    httpGet: {
      path: path,
      port: "api",
    },
    initialDelaySeconds: 30,
    periodSeconds: 5,
  };
}

const deployment = new k8s.apps.v1.Deployment(appName, {
  kind: "Deployment",
  apiVersion: "apps/v1",
  metadata: {
    name: appName,
    namespace: data.namespace,
  },
  spec: {
    replicas: data.deployment.replicas,
    selector: { matchLabels: appLabels },
    template: {
      metadata: { labels: appLabels },
      spec: {
        containers: [
          {
            name: appName,
            image: `ghcr.io/shukawam/${appName}:latest`,
            imagePullPolicy: "IfNotPresent",
            ports: [{ name: "api", containerPort: 8080 }],
            readinessProbe: defaultProbeGenerator("/health/ready"),
            livenessProbe: defaultProbeGenerator("/health/live"),
            env: [{ name: "cowweb.message", value: data.cowweb.message }],
          },
        ],
        imagePullSecrets: [{ name: "ghcr-secret" }],
      },
    },
  },
});

const service = new k8s.core.v1.Service(appName, {
  kind: "Service",
  apiVersion: "v1",
  metadata: {
    name: appName,
    namespace: data.namespace,
    labels: {
      app: appName,
      "prometheus.io/scrape": "true",
    },
  },
  spec: {
    type: "ClusterIP",
    selector: appLabels,
    ports: [{ port: 8080, targetPort: 8080, name: "http" }],
  },
});

const ingress = new k8s.networking.v1.Ingress(appName, {
  kind: "Ingress",
  apiVersion: "networking.k8s.io/v1",
  metadata: {
    name: appName,
    namespace: data.namespace,
    annotations: {
      "kubernetes.io/ingress.class": "nginx",
      "cert-manager.io/cluster-issuer": "letsencrypt-prod",
    },
  },
  spec: {
    tls: [
      {
        hosts: data.ingress.tls.hosts,
        secretName: data.ingress.tls.secretName,
      },
    ],
    rules: [
      {
        host: data.ingress.tls.hosts[0],
        http: {
          paths: [
            {
              backend: {
                service: {
                  name: appName,
                  port: {
                    number: 8080,
                  },
                },
              },
              pathType: "Prefix",
              path: "/",
            },
          ],
        },
      },
    ],
  },
});

export const deploymentName = deployment.metadata.name;
export const serviceName = service.metadata.name;
export const ingressName = ingress.metadata.name;
```

まずは、この部分です。

```ts
interface Data {
  namespace: string;
  deployment: {
    replicas: number;
  };
  ingress: {
    host: string;
    tls: {
      hosts: string[];
      secretName: string;
    };
  };
  cowweb: {
    message: string;
  };
}

const config = new pulumi.Config();
const data = config.requireObject<Data>("data");
```

環境ごとに変更する可能性のある箇所を Pulumi の Config として扱っています。Config ファイルは、プロジェクトのルートディレクトリに `Pulumi.<stack-name>.yaml` として配置します。今回の例だと中身は、以下の通り。

```Pulumi.dev.yaml
config:
  k8s-helidon-app:data:
    namespace: dev
    deployment:
      replicas: 1
    ingress:
      host: helidon.dev.shukawam.me
      tls:
        hosts:
          - helidon.dev.shukawam.me
        secretName: shukawam-tls-secret
    cowweb:
      message: Dev!
```

```Pulumi.prod.yaml
config:
  k8s-helidon-app:data:
    namespace: prod
    deployment:
      replicas: 3
    ingress:
      host: helidon.prod.shukawam.me
      tls:
        hosts:
          - helidon.prod.shukawam.me
        secretName: shukawam-tls-secret
    cowweb:
      message: Prod!
```

後は、これを TypeScript として扱うための interface 定義と専用の API (e.g. `new pulumi.Config()requireObject<Data>("data")`)を通して扱います。

ちなみに、Pulumi ではこの独立した単位を Stack として扱っています。今回は、最初に `dev` の Stack を作成しましたが、新しく `prod` の Stack を追加する際には以下のように実行します。

```bash
pulumi stack init prod
```

確認してみると、確かに Stack が 2 つ存在することが分かります。

```bash
pulumi stack ls
NAME   LAST UPDATE  RESOURCE COUNT  URL
dev    1 hour ago   5               https://app.pulumi.com/shukawam/k8s-helidon-app/dev
prod*  1 hour ago   5               https://app.pulumi.com/shukawam/k8s-helidon-app/prod
```

次に、この部分です。

```ts
function defaultProbeGenerator(path: string) {
  return {
    httpGet: {
      path: path,
      port: "api",
    },
    initialDelaySeconds: 30,
    periodSeconds: 5,
  };
}

const deployment = new k8s.apps.v1.Deployment(appName, {
  kind: "Deployment",
  apiVersion: "apps/v1",
  metadata: {
    name: appName,
    namespace: data.namespace,
  },
  spec: {
    replicas: data.deployment.replicas,
    selector: { matchLabels: appLabels },
    template: {
      metadata: { labels: appLabels },
      spec: {
        containers: [
          {
            name: appName,
            image: `ghcr.io/shukawam/${appName}:latest`,
            imagePullPolicy: "IfNotPresent",
            ports: [{ name: "api", containerPort: 8080 }],
            readinessProbe: defaultProbeGenerator("/health/ready"),
            livenessProbe: defaultProbeGenerator("/health/live"),
            env: [{ name: "cowweb.message", value: data.cowweb.message }],
          },
        ],
        imagePullSecrets: [{ name: "ghcr-secret" }],
      },
    },
  },
});
```

Kubernetes - Deployment の定義です。基本的に YAML で表現するのと大差ないのですが、readinessProbe, livenessProbe 周りを共通化したり、テンプレートリテラル(`ghcr.io/shukawam/${appName}:latest`)を使って効率的かつ簡単に実装できるのは流石、GPL のメリットというところでしょうか。環境によって、差分のある箇所は前述の通り Config を通して扱っていて、Deployment 定義だと `.metadata.namespace`, `.spec.replicas`, `.spec.template.spec.containers[0].env[0]` なんかが該当します。

Kubernetes - Service/Ingress 部分は上の説明とかなり重複するので割愛します。

実際に、プロビジョニングしてみます。まずは、作成されるインフラ構成をプレビューします。（Terraform でいうところの `terraform plan` に相当）

```bash
pulumi preview
Previewing update (dev)

View in Browser (Ctrl+O): https://app.pulumi.com/shukawam/k8s-helidon-app/dev/previews/67c4080b-ee7f-46c1-b681-c895875f9bcc

     Type                                        Name                 Plan
 +   pulumi:pulumi:Stack                         k8s-helidon-app-dev  create
 +   ├─ kubernetes:networking.k8s.io/v1:Ingress  k8s-helidon-app      create
 +   ├─ kubernetes:apps/v1:Deployment            k8s-helidon-app      create
 +   └─ kubernetes:core/v1:Service               k8s-helidon-app      create


Outputs:
    deploymentName: "k8s-helidon-app"
    ingressName   : "k8s-helidon-app"
    serviceName   : "k8s-helidon-app"

Resources:
    + 4 to create
```

`dev`の Stack に紐づき Ingress, Deployment, Service が新たに追加されることが確認できましたので、実際に作成してみます。（Terraform でいうところの `terraform apply` 相当）

```bash
pulumi up
Previewing update (dev)

View in Browser (Ctrl+O): https://app.pulumi.com/shukawam/k8s-helidon-app/dev/previews/667c0f36-78dc-4d51-8d15-17edf4ff7607

     Type                                        Name                 Plan
 +   pulumi:pulumi:Stack                         k8s-helidon-app-dev  create
 +   ├─ kubernetes:networking.k8s.io/v1:Ingress  k8s-helidon-app      create
 +   ├─ kubernetes:apps/v1:Deployment            k8s-helidon-app      create
 +   └─ kubernetes:core/v1:Service               k8s-helidon-app      create


Outputs:
    deploymentName: "k8s-helidon-app"
    ingressName   : "k8s-helidon-app"
    serviceName   : "k8s-helidon-app"

Resources:
    + 4 to create

Do you want to perform this update? yes
Updating (dev)

View in Browser (Ctrl+O): https://app.pulumi.com/shukawam/k8s-helidon-app/dev/updates/3

     Type                                        Name                 Status
 +   pulumi:pulumi:Stack                         k8s-helidon-app-dev  created (45s)
 +   ├─ kubernetes:networking.k8s.io/v1:Ingress  k8s-helidon-app      created (42s)
 +   ├─ kubernetes:apps/v1:Deployment            k8s-helidon-app      created (36s)
 +   └─ kubernetes:core/v1:Service               k8s-helidon-app      created (11s)


Outputs:
    deploymentName: "k8s-helidon-app"
    ingressName   : "k8s-helidon-app"
    serviceName   : "k8s-helidon-app"

Resources:
    + 4 created

Duration: 47s
```

確認してみます。

```bash
kubectl --namespace dev get deployment,service,ingress
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-helidon-app   1/1     1            1           6m52s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/k8s-helidon-app   ClusterIP   10.96.244.218   <none>        8080/TCP   6m52s

NAME                                        CLASS    HOSTS                     ADDRESS           PORTS     AGE
ingress.networking.k8s.io/k8s-helidon-app   <none>   helidon.dev.shukawam.me   129.153.126.126   80, 443   6m53s
```

prod の Stack についても同様に実施します。まずは、Stack を切り替えます。

```bash
pulumi stack select prod
```

作成される環境をプレビューします。

```bash
pulumi preview
Previewing update (prod)

View in Browser (Ctrl+O): https://app.pulumi.com/shukawam/k8s-helidon-app/prod/previews/bd788219-4142-4e78-b316-a77ec0ae0a6f

     Type                                        Name                  Plan
 +   pulumi:pulumi:Stack                         k8s-helidon-app-prod  create
 +   ├─ kubernetes:apps/v1:Deployment            k8s-helidon-app       create
 +   ├─ kubernetes:networking.k8s.io/v1:Ingress  k8s-helidon-app       create
 +   └─ kubernetes:core/v1:Service               k8s-helidon-app       create


Outputs:
    deploymentName: "k8s-helidon-app"
    ingressName   : "k8s-helidon-app"
    serviceName   : "k8s-helidon-app"

Resources:
    + 4 to create
```

作成します。

```bash
pulumi up
Previewing update (prod)

View in Browser (Ctrl+O): https://app.pulumi.com/shukawam/k8s-helidon-app/prod/previews/9a3b89ab-e49d-42a5-bd16-9d3d5224c410

     Type                                        Name                  Plan
 +   pulumi:pulumi:Stack                         k8s-helidon-app-prod  create
 +   ├─ kubernetes:apps/v1:Deployment            k8s-helidon-app       create
 +   ├─ kubernetes:networking.k8s.io/v1:Ingress  k8s-helidon-app       create
 +   └─ kubernetes:core/v1:Service               k8s-helidon-app       create


Outputs:
    deploymentName: "k8s-helidon-app"
    ingressName   : "k8s-helidon-app"
    serviceName   : "k8s-helidon-app"

Resources:
    + 4 to create

Do you want to perform this update? yes
Updating (prod)

View in Browser (Ctrl+O): https://app.pulumi.com/shukawam/k8s-helidon-app/prod/updates/14

     Type                                        Name                  Status
 +   pulumi:pulumi:Stack                         k8s-helidon-app-prod  created (39s)
 +   ├─ kubernetes:apps/v1:Deployment            k8s-helidon-app       created (36s)
 +   ├─ kubernetes:networking.k8s.io/v1:Ingress  k8s-helidon-app       created (7s)
 +   └─ kubernetes:core/v1:Service               k8s-helidon-app       created (11s)


Outputs:
    deploymentName: "k8s-helidon-app"
    ingressName   : "k8s-helidon-app"
    serviceName   : "k8s-helidon-app"

Resources:
    + 4 created

Duration: 41s
```

確認してみます。

```bash
kubectl --namespace prod get deployment,service,ingress
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-helidon-app   3/3     3            3           82s

NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/k8s-helidon-app   ClusterIP   10.96.26.215   <none>        8080/TCP   83s

NAME                                        CLASS    HOSTS                      ADDRESS           PORTS     AGE
ingress.networking.k8s.io/k8s-helidon-app   <none>   helidon.prod.shukawam.me   129.153.126.126   80, 443   83s
```

dev, prod の作成された結果を見比べてみると、レプリカ数が違っていたり（dev: 1, prod: 3） Ingress のホスト名が異なることが確認できます。

実際にリクエストを送って確認してみると、レスポンスの結果も異なることが確認できました。

```bash
# Stack: dev
curl https://helidon.dev.shukawam.me/cowsay/say
 ______
< Dev! >
 ------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||

# Stack: prod
curl https://helidon.prod.shukawam.me/cowsay/say
 _______
< Prod! >
 -------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

# おわりに

今回は Pulumi, [Kubernetes Provider](https://github.com/pulumi/pulumi-kubernetes) を使って簡単なアプリケーションを K8s 上（OKE を使用）に構築してみましたが、何だか Kustomize と似たようなことをやっているなあ、と思いながら実装してました。まだ、Custom Resource の扱い方が個人的に謎だったり GPL のエコシステム（テストライブラリ、IDE、等）も十分に活かせていない感じがするので、これから要研究です。

# 参考情報

使ったコードはこちらから参照ください。

https://github.com/shukawam/k8s-helidon-app/tree/main
