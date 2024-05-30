---
title: Kubernetes のオートスケーリングを OKE で試してみた
tags:
  - kubernetes
  - oci
private: false
updated_at: '2024-05-21T16:11:57+09:00'
id: 15f1aa5948dbb950368b
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

Kubernetes のオートスケーリングに関して、あまり動かす機会がなかったのでこの機にまとめて取り組んでみようかと思います。

# Kubernetes とオートスケーリング

Kubernetes に限らず、負荷状況に応じた適切なリソース配置やインフラのコスト管理を自動的に行うために、オートスケーリングに取り組むことがあると思います。
Kubernetes では、以下のスケール手法が主に使われます。

| 種類                           | 対象のリソース | スケールの方向 | 概要                                                                      |
| ------------------------------ | -------------- | -------------- | ------------------------------------------------------------------------- |
| Horizontal Pod Autoscaler(HPA) | Pod            | 水平方向       | 負荷の増減に対応するために Pod の数を自動的に増減する                     |
| Vertical Pod Autoscaler(VPA)   | Pod            | 垂直方向       | 負荷の増減に対応するために Pod に設定されたリソースリクエストを上書きする |
| Cluster Autoscaler(CA)         | Node           | 水平方向       | 負荷の増減に対応するためにクラスタを構成するノードの台数を調整する        |

:::note warn

上記であげたスケール手法以外にも、[Multi-dimensional Pod Autoscaler](https://github.com/kubernetes/autoscaler/blob/master/multidimensional-pod-autoscaler/AEP.md)(MPA)と呼ばれる水平垂直スケールする仕組みがコミュニティで検討されていたり、[Karpentar](https://karpenter.sh/) などのオートスケールに用いられるツールなどが存在しますが、本記事ではスコープ外とさせていただきます。

:::

# Walk through w/ OKE

今回は、Oracle Container Engine for Kubernetes(OKE)を用いて、上述した Kubernetes におけるオートスケーリング手法(HPA, VPA, CA)を一通り試してみます。

## 前提

本記事は、Kubernetes v1.29.1 にて動作確認を行いました。また、HPA, VPA で必要な [metrics-server](https://github.com/kubernetes-sigs/metrics-server/) は事前に Kubernetes クラスタに対してデプロイ済みなものとします。

## HPA

https://docs.oracle.com/ja-jp/iaas/Content/ContEng/Tasks/contengusinghorizontalpodautoscaler.htm

に基づいて進めていきます。まずは、metrics-server がデプロイされていることを確認します。

```sh
$ kubectl -n kube-system get deploy metrics-server
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           37h
```

スケール対象となるサンプルワークロードをデプロイします。

```sh
kubectl -n example apply -f https://k8s.io/examples/application/php-apache.yaml
```

中身は以下のようになっています。(`resources.requests.cpu: 200m`, `resources.limits.cpu: 500m`)

```php-apache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
```

このアプリケーションに対して、平均の CPU 使用率が 50%を維持するように Pod の数を 1~10 個の間で調整する HPA リソースを以下のように定義します。

```php-apache-hpa.yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  targetCPUUtilizationPercentage: 50
```

これもクラスタにデプロイしておきます。

```sh
kubectl -n example apply -f php-apache-hpa.yaml
```

HPA リソースを確認してみると、以下のようになっています。

```sh
$ kubectl -n example get hpa -w
NAME             REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache-hpa   Deployment/php-apache   0%/50%    1         10        1          87s
```

対象のアプリケーションに対して、別の Pod から負荷をかけていきます。

```sh
$ kubectl -n example run -it --rm load-generator --image=busybox -- /bin/sh
/ # while true; do wget -q -O- http://php-apache; done
OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!, ...
```

php-apache に対して、負荷がかかると HPA リソースが平均の CPU 使用率を 50%に維持するためにレプリカ数を増やしていることを確認することができます。

```sh
NAME             REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache-hpa   Deployment/php-apache   0%/50%    1         10        1          30m
php-apache-hpa   Deployment/php-apache   18%/50%   1         10        1          30m
php-apache-hpa   Deployment/php-apache   250%/50%   1         10        1          30m
php-apache-hpa   Deployment/php-apache   249%/50%   1         10        4          31m
php-apache-hpa   Deployment/php-apache   99%/50%    1         10        5          31m
php-apache-hpa   Deployment/php-apache   88%/50%    1         10        5          31m
php-apache-hpa   Deployment/php-apache   70%/50%    1         10        5          31m
# ... 省略 ...
```

## VPA

https://docs.oracle.com/ja-jp/iaas/Content/ContEng/Tasks/contengusingverticalpodautoscaler.htm

に基づいて進めていきます。まずは、VPA を実施するために必要なリソースをクラスタ上に構築します。Autoscaler のソースコードをダウンロードします。

```sh
git clone https://github.com/kubernetes/autoscaler.git
```

次に、コード内で提供されているスクリプトを用いて必要なリソースをデプロイします。

```sh
cd autoscaler/vertical-pod-autoscaler; ./hack/vpa-up.sh
```

VPA で必要なリソースがデプロイされていることを確認します。

```sh
$ kubectl -n example get pods | grep -i vpa
vpa-admission-controller-548b84c8c8-vd49f   1/1     Running   0             37h
vpa-recommender-8f4b9d68c-bjk6j             1/1     Running   0             37h
vpa-updater-5c88cd9cb6-xpcs8                1/1     Running   0             37h
```

デプロイされている各リソースの概要は以下の通りです。

| 名前                                   | 概要                                                                                                                                            |
| -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Recommender                            | リソースの使用量をモニターし、コンテナに推奨される CPU、メモリのリクエスト値（`resources.requests.cpu`, `resources.requests.memory`）を提供する |
| Updater                                | リソースが正しくない Pod をチェックして削除し、更新されたリクエスト値で Pod を再作成できるようにする                                            |
| Admission Plugin(Admission Controller) | 新しい Pod に正しいリソース・リクエスト値を設定する                                                                                             |

VPA 用のサンプルアプリケーションをデプロイします。

```sh
kubectl -n example apply -f examples/hamster.yaml
```

現在、垂直スケール時には、Pod の再起動が発生するため起動された Pod に割り当てられたリソース・リクエストを参照するために Pod の状態を watch しておきます。

```sh
$ kubectl -n example get pods -l app=hamster -w
NAME                      READY   STATUS    RESTARTS   AGE
hamster-c6967774f-6kmz8   1/1     Running   0          9s
hamster-c6967774f-fgb8c   1/1     Running   0          9s
hamster-c6967774f-fgb8c   1/1     Running   0          75s
hamster-c6967774f-fgb8c   1/1     Terminating   0          75s
hamster-c6967774f-fgb8c   1/1     Terminating   0          75s
hamster-c6967774f-67dhv   0/1     Pending       0          0s
hamster-c6967774f-67dhv   0/1     Pending       0          0s
hamster-c6967774f-67dhv   0/1     ContainerCreating   0          0s
hamster-c6967774f-67dhv   1/1     Running             0          2s # これ
```

新しく作成された Pod のリソース・リクエストを参照してみます。

```sh
kubectl -n example describe pods hamster-c6967774f-67dhv
```

実行結果

```yaml
Requests:
  cpu: 587m
  memory: 262144k
```

必要なリソース量が計算され、それに上書きされ新しく Pod が起動していることを確認することができました。

ちなみに、`hamster.yaml` は以下のようになっています。

```hamster.yaml
# This config creates a deployment with two pods, each requesting 100 millicores
# and trying to utilize slightly above 500 millicores (repeatedly using CPU for
# 0.5s and sleeping 0.5s).
# It also creates a corresponding Vertical Pod Autoscaler that adjusts the
# requests.
# Note that the update mode is left unset, so it defaults to "Auto" mode.
---
apiVersion: "autoscaling.k8s.io/v1"
kind: VerticalPodAutoscaler
metadata:
  name: hamster-vpa
spec:
  # recommenders field can be unset when using the default recommender.
  # When using an alternative recommender, the alternative recommender's name
  # can be specified as the following in a list.
  # recommenders:
  #   - name: 'alternative'
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: hamster
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 100m
          memory: 50Mi
        maxAllowed:
          cpu: 1
          memory: 500Mi
        controlledResources: ["cpu", "memory"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hamster
spec:
  selector:
    matchLabels:
      app: hamster
  replicas: 2
  template:
    metadata:
      labels:
        app: hamster
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534 # nobody
      containers:
        - name: hamster
          image: registry.k8s.io/ubuntu-slim:0.1
          resources:
            requests:
              cpu: 100m
              memory: 50Mi
          command: ["/bin/sh"]
          args:
            - "-c"
            - "while true; do timeout 0.5s yes >/dev/null; sleep 0.5s; done"
```

`resourcePolicy` で記載されている通り、100 ミリコア ~ 1 コア、50Mi ~ 500Mi の間に垂直にスケールします。

```yaml
resourcePolicy:
  containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 100m
        memory: 50Mi
      maxAllowed:
        cpu: 1
        memory: 500Mi
      controlledResources: ["cpu", "memory"]
```

Deployment の Resource Requests を参照してみると、以下のように設定されていますが CPU もメモリも意図的にリソース不足になるように設定されているみたいです。

```yaml
resources:
  requests:
    cpu: 100m
    memory: 50Mi
```

## CA

https://docs.oracle.com/ja-jp/iaas/Content/ContEng/Tasks/contengusingclusterautoscaler_topic-Working_with_the_Cluster_Autoscaler.htm

に基づいて進めていきます。OKE クラスタからノード・プールに対してアクションをするために必要な IAM ポリシーの設定は割愛させていただきます。また、OKE クラスタは以下のようにオートスケールの対象外のノード・プールと対象のノード・プールが設定されていることを前提として進めます。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/7905b927-8916-d185-6284-92dcfde969b9.png)

まずは、CA を実現するために必要なリソースをクラスタにデプロイします。

```cluster-autoscaler.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
  name: cluster-autoscaler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["events", "endpoints"]
    verbs: ["create", "patch"]
  - apiGroups: [""]
    resources: ["pods/eviction"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["pods/status"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["endpoints"]
    resourceNames: ["cluster-autoscaler"]
    verbs: ["get", "update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["watch", "list", "get", "patch", "update"]
  - apiGroups: [""]
    resources:
      - "pods"
      - "services"
      - "replicationcontrollers"
      - "persistentvolumeclaims"
      - "persistentvolumes"
    verbs: ["watch", "list", "get"]
  - apiGroups: ["extensions"]
    resources: ["replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["policy"]
    resources: ["poddisruptionbudgets"]
    verbs: ["watch", "list"]
  - apiGroups: ["apps"]
    resources: ["statefulsets", "replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses", "csinodes"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["batch", "extensions"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["create"]
  - apiGroups: ["coordination.k8s.io"]
    resourceNames: ["cluster-autoscaler"]
    resources: ["leases"]
    verbs: ["get", "update"]
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["watch", "list"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["csidrivers", "csistoragecapacities"]
    verbs: ["watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["create","list","watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["cluster-autoscaler-status", "cluster-autoscaler-priority-expander"]
    verbs: ["delete", "get", "update", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '8085'
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
        - image: lhr.ocir.io/oracle/oci-cluster-autoscaler:1.29.0-10
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 300Mi
            requests:
              cpu: 100m
              memory: 300Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=oci
            - --max-node-provision-time=25m
            - --nodes=1:2:ocid1.nodepool.oc1.uk-london-1.aaaaaaaaaqqfcvolpdrndsvgp4fnyly52sntduhbt7gkt5guynhezlqk2kla
            - --scale-down-delay-after-add=10m
            - --scale-down-unneeded-time=10m
            - --unremovable-node-recheck-timeout=5m
            - --balance-similar-node-groups
            - --balancing-ignore-label=displayName
            - --balancing-ignore-label=hostname
            - --balancing-ignore-label=internal_addr
            - --balancing-ignore-label=oci.oraclecloud.com/fault-domain
          imagePullPolicy: "Always"
          env:
          - name: OKE_USE_INSTANCE_PRINCIPAL
            value: "true"
          - name: OCI_SDK_APPEND_USER_AGENT
            value: "oci-oke-cluster-autoscaler"

```

ServiceAccount, ClusterRole, Role, ClusterRoleBinding, RoleBinding は置いておいて、重要なのは Deployment に設定しているパラメータ群です。とりあえずの動作確認に必要な箇所のみに絞って解説します。

まずは、イメージ名です。CA の実態は Deployment なのでデプロイするコンテナイメージを指定する必要があります。これは、OCI のリージョンと Kubernetes のバージョンによって決定されるもので、どのイメージを使えば良いかは[ステップ 2: Cluster Autoscaler 構成ファイルのコピーおよびカスタマイズ](https://docs.oracle.com/ja-jp/iaas/Content/ContEng/Tasks/contengusingclusterautoscaler_topic-Working_with_the_Cluster_Autoscaler.htm#contengusingclusterautoscaler_topic-Working_with_the_Cluster_Autoscaler-step-copy-CA-config-file)に記載があります。

```yaml
image: lhr.ocir.io/oracle/oci-cluster-autoscaler:1.29.0-10
```

次に、CA の対象となるノード・プールを指定します。今回は、指定のノード・プール内で負荷に応じて最大 2 台のノードがプロビジョニングされるように設定してみました。

```yaml
- --nodes=1:2:ocid1.nodepool.oc1.uk-london-1.aaaaaaaaaqqfcvolpdrndsvgp4fnyly52sntduhbt7gkt5guynhezlqk2kla
```

変更後に、CA 用の関連リソースをデプロイします。

```sh
kubectl apply -f cluster-autoscaler.yaml
```

Cluster Autoscaler が正常に動作しているかどうかを確認します。

```sh
$ kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler
# ... 省略 ...
I0521 06:34:08.169007       1 static_autoscaler.go:642] Starting scale down
I0521 06:34:18.182579       1 static_autoscaler.go:290] Starting main loop
I0521 06:34:18.183085       1 oci_manager.go:451] did not find node pool for reference: {AvailabilityDomain:UK-LONDON-1-AD-1 Name:10.0.10.230 CompartmentID:ocid1.compartment.oc1..aaaaaaaac5t2rwhyzq6fm6pepwdn6gc434ymir7sgvh4ac4sd5sre4altoka InstanceID:ocid1.instance.oc1.uk-london-1.anwgiljrssl65iqc2kmgxagkfepoxgts4cfq53o7n5wq7mg3fbn737h5iqcq NodePoolID:ocid1.nodepool.oc1.uk-london-1.aaaaaaaarbie6tyvecy7jgyhybjv5tbjjja2fomiexkfgobn3nasgahvhftq InstancePoolID: PrivateIPAddress:10.0.10.230 PublicIPAddress: Shape:VM.Standard.E4.Flex}
I0521 06:34:18.183443       1 oci_manager.go:451] did not find node pool for reference: {AvailabilityDomain:UK-LONDON-1-AD-1 Name:10.0.10.230 CompartmentID:ocid1.compartment.oc1..aaaaaaaac5t2rwhyzq6fm6pepwdn6gc434ymir7sgvh4ac4sd5sre4altoka InstanceID:ocid1.instance.oc1.uk-london-1.anwgiljrssl65iqc2kmgxagkfepoxgts4cfq53o7n5wq7mg3fbn737h5iqcq NodePoolID:ocid1.nodepool.oc1.uk-london-1.aaaaaaaarbie6tyvecy7jgyhybjv5tbjjja2fomiexkfgobn3nasgahvhftq InstancePoolID: PrivateIPAddress:10.0.10.230 PublicIPAddress: Shape:VM.Standard.E4.Flex}
I0521 06:34:18.183555       1 filter_out_schedulable.go:63] Filtering out schedulables
I0521 06:34:18.183569       1 filter_out_schedulable.go:120] 0 pods marked as unschedulable can be scheduled.
I0521 06:34:18.183586       1 filter_out_schedulable.go:83] No schedulable pods
I0521 06:34:18.183592       1 filter_out_daemon_sets.go:40] Filtering out daemon set pods
I0521 06:34:18.183597       1 filter_out_daemon_sets.go:49] Filtered out 0 daemon set pods, 0 unschedulable pods left
I0521 06:34:18.183643       1 static_autoscaler.go:547] No unschedulable pods
I0521 06:34:18.183671       1 static_autoscaler.go:570] Calculating unneeded nodes
I0521 06:34:18.183682       1 oci_manager.go:451] did not find node pool for reference: {AvailabilityDomain:UK-LONDON-1-AD-1 Name:10.0.10.230 CompartmentID:ocid1.compartment.oc1..aaaaaaaac5t2rwhyzq6fm6pepwdn6gc434ymir7sgvh4ac4sd5sre4altoka InstanceID:ocid1.instance.oc1.uk-london-1.anwgiljrssl65iqc2kmgxagkfepoxgts4cfq53o7n5wq7mg3fbn737h5iqcq NodePoolID:ocid1.nodepool.oc1.uk-london-1.aaaaaaaarbie6tyvecy7jgyhybjv5tbjjja2fomiexkfgobn3nasgahvhftq InstancePoolID: PrivateIPAddress:10.0.10.230 PublicIPAddress: Shape:VM.Standard.E4.Flex}
I0521 06:34:18.183719       1 pre_filtering_processor.go:57] Node 10.0.10.230 should not be processed by cluster autoscaler (no node group config)
I0521 06:34:18.183731       1 pre_filtering_processor.go:67] Skipping 10.0.10.245 - node group min size reached (current: 1, min: 1)
I0521 06:34:18.183810       1 static_autoscaler.go:617] Scale down status: lastScaleUpTime=2024-05-21 01:32:45.031796054 +0000 UTC m=-3579.148444691 lastScaleDownDeleteTime=2024-05-21 02:43:00.955313331 +0000 UTC m=+636.775072576 lastScaleDownFailTime=2024-05-21 01:32:45.031796054 +0000 UTC m=-3579.148444691 scaleDownForbidden=false scaleDownInCooldown=false
I0521 06:34:18.183893       1 static_autoscaler.go:642] Starting scale down
I0521 06:34:26.187926       1 reflector.go:800] k8s.io/client-go/informers/factory.go:159: Watch close - *v1.CSIStorageCapacity total 7 items received
I0521 06:34:26.271998       1 reflector.go:800] k8s.io/client-go/informers/factory.go:159: Watch close - *v1.Pod total 0 items received
I0521 06:34:28.196453       1 static_autoscaler.go:290] Starting main loop
I0521 06:34:28.197010       1 oci_manager.go:451] did not find node pool for reference: {AvailabilityDomain:UK-LONDON-1-AD-1 Name:10.0.10.230 CompartmentID:ocid1.compartment.oc1..aaaaaaaac5t2rwhyzq6fm6pepwdn6gc434ymir7sgvh4ac4sd5sre4altoka InstanceID:ocid1.instance.oc1.uk-london-1.anwgiljrssl65iqc2kmgxagkfepoxgts4cfq53o7n5wq7mg3fbn737h5iqcq NodePoolID:ocid1.nodepool.oc1.uk-london-1.aaaaaaaarbie6tyvecy7jgyhybjv5tbjjja2fomiexkfgobn3nasgahvhftq InstancePoolID: PrivateIPAddress:10.0.10.230 PublicIPAddress: Shape:VM.Standard.E4.Flex}
I0521 06:34:28.197290       1 oci_manager.go:451] did not find node pool for reference: {AvailabilityDomain:UK-LONDON-1-AD-1 Name:10.0.10.230 CompartmentID:ocid1.compartment.oc1..aaaaaaaac5t2rwhyzq6fm6pepwdn6gc434ymir7sgvh4ac4sd5sre4altoka InstanceID:ocid1.instance.oc1.uk-london-1.anwgiljrssl65iqc2kmgxagkfepoxgts4cfq53o7n5wq7mg3fbn737h5iqcq NodePoolID:ocid1.nodepool.oc1.uk-london-1.aaaaaaaarbie6tyvecy7jgyhybjv5tbjjja2fomiexkfgobn3nasgahvhftq InstancePoolID: PrivateIPAddress:10.0.10.230 PublicIPAddress: Shape:VM.Standard.E4.Flex}
I0521 06:34:28.197335       1 filter_out_schedulable.go:63] Filtering out schedulables
I0521 06:34:28.197359       1 filter_out_schedulable.go:120] 0 pods marked as unschedulable can be scheduled.
I0521 06:34:28.197368       1 filter_out_schedulable.go:83] No schedulable pods
I0521 06:34:28.197373       1 filter_out_daemon_sets.go:40] Filtering out daemon set pods
I0521 06:34:28.197377       1 filter_out_daemon_sets.go:49] Filtered out 0 daemon set pods, 0 unschedulable pods left
I0521 06:34:28.197449       1 static_autoscaler.go:547] No unschedulable pods
I0521 06:34:28.197471       1 static_autoscaler.go:570] Calculating unneeded nodes
I0521 06:34:28.197497       1 oci_manager.go:451] did not find node pool for reference: {AvailabilityDomain:UK-LONDON-1-AD-1 Name:10.0.10.230 CompartmentID:ocid1.compartment.oc1..aaaaaaaac5t2rwhyzq6fm6pepwdn6gc434ymir7sgvh4ac4sd5sre4altoka InstanceID:ocid1.instance.oc1.uk-london-1.anwgiljrssl65iqc2kmgxagkfepoxgts4cfq53o7n5wq7mg3fbn737h5iqcq NodePoolID:ocid1.nodepool.oc1.uk-london-1.aaaaaaaarbie6tyvecy7jgyhybjv5tbjjja2fomiexkfgobn3nasgahvhftq InstancePoolID: PrivateIPAddress:10.0.10.230 PublicIPAddress: Shape:VM.Standard.E4.Flex}
I0521 06:34:28.197510       1 pre_filtering_processor.go:57] Node 10.0.10.230 should not be processed by cluster autoscaler (no node group config)
I0521 06:34:28.197519       1 pre_filtering_processor.go:67] Skipping 10.0.10.245 - node group min size reached (current: 1, min: 1)
I0521 06:34:28.197573       1 static_autoscaler.go:617] Scale down status: lastScaleUpTime=2024-05-21 01:32:45.031796054 +0000 UTC m=-3579.148444691 lastScaleDownDeleteTime=2024-05-21 02:43:00.955313331 +0000 UTC m=+636.775072576 lastScaleDownFailTime=2024-05-21 01:32:45.031796054 +0000 UTC m=-3579.148444691 scaleDownForbidden=false scaleDownInCooldown=false
# ... 省略 ...
```

のようなログが継続的に出力されていれば OK です。ここから、サンプルワークロードをデプロイする前にノード数がスケールすることを確認するためにノードの状況を watch しておきます。

```sh
$ kubectl get nodes -w
NAME          STATUS   ROLES   AGE   VERSION
10.0.10.230   Ready    node    14d   v1.29.1
10.0.10.245   Ready    node    21h   v1.29.1
```

次に、サンプルワークロードをデプロイします。今回は 100 個のレプリカ数を持つ Nginx の Deployment とします。

```sh
kubectl -n create deployment nginx-deployment --replicas=100 --image nginx:latest
```

Deployment の状況を watch しておきます。

```sh
kubectl -n example get deployment nginx-deployment -w
```

しばらく経過すると、Pod をプロビジョニングするためにノード数が増えていることを確認できます。（OCI 内で Compute Instance のプロビジョンングが行われるため、そこそこ時間がかかります）

```sh
NAME          STATUS   ROLES   AGE   VERSION
10.0.10.230   Ready    node    14d   v1.29.1
10.0.10.245   Ready    node    21h   v1.29.1
10.0.10.245   Ready    node    21h   v1.29.1
10.0.10.3     NotReady   <none>   0s    v1.29.1
10.0.10.3     NotReady   <none>   0s    v1.29.1
10.0.10.3     NotReady   <none>   0s    v1.29.1
10.0.10.3     NotReady   <none>   0s    v1.29.1
10.0.10.3     NotReady   <none>   2s    v1.29.1
10.0.10.3     NotReady   <none>   10s   v1.29.1
10.0.10.3     NotReady   <none>   31s   v1.29.1
10.0.10.3     NotReady   node     34s   v1.29.1
10.0.10.230   Ready      node     14d   v1.29.1
10.0.10.3     NotReady   node     2m3s   v1.29.1
10.0.10.3     NotReady   node     2m13s   v1.29.1
10.0.10.3     NotReady   node     2m13s   v1.29.1
10.0.10.245   Ready      node     21h     v1.29.1
10.0.10.3     Ready      node     2m34s   v1.29.1
# ... 省略 ...
```

# 終わりに

細かいことはさておき Kubernetes で使われるオートスケーリングの手法について OKE を用いて試してみました。

# 参考

https://thinkit.co.jp/article/22905

https://kubernetes.io/ja/docs/tasks/run-application/horizontal-pod-autoscale/

https://kubernetes.io/ja/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/oci/README.md
