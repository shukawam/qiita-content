---
title: Pulumi を用いた OCI Compute の作成（+簡易的な初期セットアップ）
tags:
  - oci
  - Pulumi
private: false
updated_at: '2023-12-01T15:52:53+09:00'
id: 035fa4e8a1a937d2cc90
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

こちらの記事は、[Oracle Cloud Infrastructure Advent Calender 2023](https://qiita.com/advent-calendar/2023/oci) Day 1 の記事として書かれています。今回は、個人的に注目している IaC ツールである [Pulumi](https://www.pulumi.com/b/) を使って普段の作業用の Compute インスタンスを構築を効率化します。

## Pulumi とは？

Pulumi は、Pulumi 社が開発しているクラウドインフラを作成・管理するための OSS の IaC ツールです。リソースのあるべき状態をコードとして定義し、Pulumi がパブリッククラウドなどのプロバイダーに対してユーザーの代わりに API リクエストを行うことでインフラのプロビジョニングを行います。似たようなツールで有名なものに Terraform がありますが、大きな違いは実装に用いる言語でしょう。Terraform は、HCL という DSL[^1] で実装するのに対し、Pulumi は、TypeScript, Golang, Python, Java といった GPL[^2] で実装します。そのため、

- 使用する言語自体を習熟している場合、ライブラリの使い方を学ぶだけで IaC のコードが実装できる
- GPL ならではの柔軟な表現力を用いて IaC のコードが実装できる
  - e.g. 分岐処理、繰り返し処理、etc.
- その言語のエコシステムの恩恵を受けることができる
  - e.g. IDE、テストライブラリ、静的解析ツール、etc.

といったメリットが存在します。また、最近は [Pulumi AI](https://www.pulumi.com/ai) という LLM(GPT) を用いて IaC のコードを自然言語から生成させる取り組みもされているようです。

[^1]: Domain Specific Language の略であるタスク向けに設計されたプログラミング言語のこと
[^2]: General Purpose Language の略で汎用的にタスクを実行することを目的に設計されたプログラミング言語のこと

## 今回作成するインフラ構成について

今回は以下のような構成を `pulumi up` のコマンド一発で作成します。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/e6796326-b0eb-8b11-56a2-342452037cb0.png)

VCN ウィザードのインターネット接続性を持つ VCN の作成で作られる VCN の中に Compute Instance を作成します。個人的な検証用なので、そこまで条件はないですが以下のような条件で作ってみます。

- VM.Standard.E.\*Flex 系のシェイプであればなんでも良い
- Canonical Ubuntu 22.0.4 系の最新イメージを使う
- 自分のユーザー作成と公開鍵を含めておく
- 最低限使うツールはセットアップしておく

## コードの解説

今回、IaC のコードは TypeScript で実装しました。

以下のコードがエントリーポイントとなります。

```index.ts
import * as pulumi from "@pulumi/pulumi";

import { Data } from "./data/data";
import { VcnProvider } from "./providers/vcn-provider";
import { InstanceProvider } from "./providers/instance-provider";
import { readFileSync } from "fs";

const config = new pulumi.Config();
const data = config.requireObject<Data>("data");

const vcnProvider = new VcnProvider(
  data.compartmentId,
  data.vcn.prefix,
  data.vcn.cidrBlock,
  data.vcn.publicSubnet,
  data.vcn.privateSubnet
);
const networkIds = vcnProvider.createNetwork();

const instanceProvider = new InstanceProvider(
  data.compartmentId,
  data.instance.prefix
);

const ad = pulumi.output(instanceProvider.listAds()).availabilityDomains[0]
  .name;
const shape = pulumi.output(instanceProvider.listShapes()).shapes[0].name;
const image = pulumi.output(instanceProvider.listImages()).images[0].id;

const userData = readFileSync('./user_data/user_data.yaml');

export const instance = instanceProvider.createInstance(
  ad,
  shape,
  data.instance.ocpus,
  data.instance.memoryInGbs,
  image,
  networkIds.publicSubnetId,
  userData,
);

```

以下の部分で、環境によっては可変となり得る部分を入力として受け取っています。

```ts
const config = new pulumi.Config();
const data = config.requireObject<Data>("data");
```

同じディレクトリに含まれている `Pulumi.prod.yaml` にパラメータ定義がされています。今回の場合は以下のように、compartmentId や、ネットワークの CIDR ブロックの設定、インスタンスのスペックなどを定義しています。

```Pulumi.prod.yaml
config:
  oci:region: ap-tokyo-1
  work-vm:data:
    compartmentId: ocid1.compartment.oc1...
    vcn:
      prefix: shukawam
      cidrBlock: "192.168.0.0/16"
      publicSubnet: "192.168.0.0/24"
      privateSubnet: "192.168.1.0/24"
    instance:
      prefix: shukawam
      ocpus: 1
      memoryInGbs: 16
```

以下の部分では、ネットワーク周りを作成しています。

```ts
const vcnProvider = new VcnProvider(
  data.compartmentId,
  data.vcn.prefix,
  data.vcn.cidrBlock,
  data.vcn.publicSubnet,
  data.vcn.privateSubnet
);
const networkIds = vcnProvider.createNetwork();
```

`VcnProvider` の中身は特に工夫点がないため割愛しますが、VCN やサブネット、セキュリティリストなどを作成するためのコードが実装されています。詳細は、リポジトリをご覧ください。

https://github.com/shukawam/pulumi-stacks/blob/main/oci/work-vm/providers/vcn-provider.ts

続いて、以下の部分でインスタンスを作成しています。

```ts
const instanceProvider = new InstanceProvider(
  data.compartmentId,
  data.instance.prefix
);

const ad = pulumi.output(instanceProvider.listAds()).availabilityDomains[0]
  .name;
const shape = pulumi.output(instanceProvider.listShapes()).shapes[0].name;
const image = pulumi.output(instanceProvider.listImages()).images[0].id;

const userData = readFileSync("./user_data/user_data.yaml");

export const instance = instanceProvider.createInstance(
  ad,
  shape,
  data.instance.ocpus,
  data.instance.memoryInGbs,
  image,
  networkIds.publicSubnetId,
  userData
);
```

AD(Availability Domain)やインスタンス・シェイプ、イメージなどは提供されている参照用のメソッドを用いて条件に合致するようなものを動的に取得しています。

```InstanceProvider.ts
  // 使用可能な AD を取得
  listAds(): Promise<GetAvailabilityDomainsResult> {
    const args: GetAvailabilityDomainsArgs = {
      compartmentId: this.compartmentId,
    };
    return getAvailabilityDomains(args);
  }

  // VM.Standard.E.*Flex 系のシェイプの一覧を取得
  listShapes(): Promise<GetShapesResult> {
    const args: GetShapesArgs = {
      compartmentId: this.compartmentId,
      filters: [
        {
          name: "name",
          values: ["VM.Standard.E.*Flex"],
          regex: true,
        },
      ],
    };
    return getShapes(args);
  }

  // Canonical Ubuntu 22.04 系のイメージのうち作成順にソートしたリストを取得
  listImages(): Promise<GetImagesResult> {
    const args: GetImagesArgs = {
      compartmentId: this.compartmentId,
      operatingSystem: "Canonical Ubuntu",
      operatingSystemVersion: "22.04",
      sortBy: "TIMECREATED",
      sortOrder: "DESC"
    };
    return getImages(args);
  }
```

また、インスタンスの初期セットアップに用いる cloud-init は `./user_data/user_data.yaml` から読み込んだものを使用しています。今回は以下の通りとなっています。ユーザーを作成したり、SSH 接続に使用する公開鍵を GitHub からインポートしたり、最低限使用するツールのセットアップ（Docker, kubectl）を行なっています。

```yaml
#cloud-config
chpasswd:
  list: |
    shukawam:ChangeMe!!
  expire: false
users:
  - default
  - name: shukawam
    lock_passwd: false
    groups: sudo, users, admin
    shell: /bin/bash
    sudo: ["ALL=(ALL) NOPASSWD=ALL"]
    ssh_import_id:
      - gh:shukawam
system_info:
  default_user:
    name: default-user
    lock_passwd: false
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
ssh_pwauth: no
random_seed:
  file: /dev/urandom
  command: ["pollinate", "-r", "-s", "https://entropy.ubuntu.com"]
  command_required: true
package_upgrade: true
packages:
  - curl
  - vim
  - git
  - unzip
  - gnupg
  - lsb-release
  - ca-certificates
  - dstat
runcmd:
  # install docker
  - curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
  - echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  - sudo apt-get update
  - sudo apt-get install -y docker-ce docker-ce-cli containerd.io
  - sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
  - sudo chmod +x /usr/local/bin/docker-compose
  # install kubectl
  - curl -LO "https://dl.k8s.io/release/$(curl -LS https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  - chmod +x ./kubectl
  - sudo mv ./kubectl /usr/local/bin/kubectl
```

## 環境の作成

実際に環境を作成してみます。

```sh
pulumi up
```

```sh
Previewing update (prod)

View in Browser (Ctrl+O): https://app.pulumi.com/shukawam/work-vm/prod/previews/9af2f8bb-82b4-45cf-ad2c-109580d26908

     Type                  Name              Plan
     pulumi:pulumi:Stack   work-vm-prod
 +   └─ oci:Core:Instance  shukawam-compute  create

Outputs:
  + instance: {
      + agentConfig                   : output<string>
      + async                         : output<string>
      + availabilityConfig            : output<string>
      + availabilityDomain            : "TGjA:AP-TOKYO-1-AD-1"
      + bootVolumeId                  : output<string>
      + capacityReservationId         : output<string>
      + compartmentId                 : "ocid1.compartment.oc1..."
      + createVnicDetails             : output<string>
      + dedicatedVmHostId             : output<string>
      + definedTags                   : output<string>
      + displayName                   : "shukawam"
      + extendedMetadata              : output<string>
      + faultDomain                   : output<string>
      + freeformTags                  : output<string>
      + hostnameLabel                 : output<string>
      + id                            : output<string>
      + image                         : output<string>
      + instanceOptions               : output<string>
      + ipxeScript                    : output<string>
      + isPvEncryptionInTransitEnabled: output<string>
      + launchMode                    : output<string>
      + launchOptions                 : output<string>
      + metadata                      : {
          + user_data: "I2Nsb3VkLWNvbmZpZwpjaHBhc3N3ZDoKICBsaXN0OiB8CiAgICBzaHVrYXdhbTpDaGFuZ2VNZSEhCiAgZXhwaXJlOiBmYWxzZQp1c2VyczoKICAtIGRlZmF1bHQKICAtIG5hbWU6IHNodWthd2FtCiAgICBsb2NrX3Bhc3N3ZDogZmFsc2UKICAgIGdyb3Vwczogc3VkbywgdXNlcnMsIGFkbWluCiAgICBzaGVsbDogL2Jpbi9iYXNoCiAgICBzdWRvOiBbIkFMTD0oQUxMKSBOT1BBU1NXRD1BTEwiXQogICAgc3NoX2ltcG9ydF9pZDoKICAgICAgLSBnaDpzaHVrYXdhbQpzeXN0ZW1faW5mbzoKICBkZWZhdWx0X3VzZXI6CiAgICBuYW1lOiBkZWZhdWx0LXVzZXIKICAgIGxvY2tfcGFzc3dkOiBmYWxzZQogICAgc3VkbzogWyJBTEw9KEFMTCkgTk9QQVNTV0Q6QUxMIl0Kc3NoX3B3YXV0aDogbm8KcmFuZG9tX3NlZWQ6CiAgZmlsZTogL2Rldi91cmFuZG9tCiAgY29tbWFuZDogWyJwb2xsaW5hdGUiLCAiLXIiLCAiLXMiLCAiaHR0cHM6Ly9lbnRyb3B5LnVidW50dS5jb20iXQogIGNvbW1hbmRfcmVxdWlyZWQ6IHRydWUKcGFja2FnZV91cGdyYWRlOiB0cnVlCnBhY2thZ2VzOgogIC0gY3VybAogIC0gdmltCiAgLSBnaXQKICAtIHVuemlwCiAgLSBnbnVwZwogIC0gbHNiLXJlbGVhc2UKICAtIGNhLWNlcnRpZmljYXRlcwogIC0gZHN0YXQKcnVuY21kOgogICMgaW5zdGFsbCBkb2NrZXIKICAtIGN1cmwgLWZzU0wgaHR0cHM6Ly9kb3dubG9hZC5kb2NrZXIuY29tL2xpbnV4L3VidW50dS9ncGcgfCBzdWRvIGdwZyAtLWRlYXJtb3IgLW8gL3Vzci9zaGFyZS9rZXlyaW5ncy9kb2NrZXItYXJjaGl2ZS1rZXlyaW5nLmdwZwogIC0gZWNobyAiZGViIFthcmNoPSQoZHBrZyAtLXByaW50LWFyY2hpdGVjdHVyZSkgc2lnbmVkLWJ5PS91c3Ivc2hhcmUva2V5cmluZ3MvZG9ja2VyLWFyY2hpdmUta2V5cmluZy5ncGddIGh0dHBzOi8vZG93bmxvYWQuZG9ja2VyLmNvbS9saW51eC91YnVudHUgJChsc2JfcmVsZWFzZSAtY3MpIHN0YWJsZSIgfCBzdWRvIHRlZSAvZXRjL2FwdC9zb3VyY2VzLmxpc3QuZC9kb2NrZXIubGlzdCA+IC9kZXYvbnVsbAogIC0gc3VkbyBhcHQtZ2V0IHVwZGF0ZQogIC0gc3VkbyBhcHQtZ2V0IGluc3RhbGwgLXkgZG9ja2VyLWNlIGRvY2tlci1jZS1jbGkgY29udGFpbmVyZC5pbwogIC0gc3VkbyBjdXJsIC1MICJodHRwczovL2dpdGh1Yi5jb20vZG9ja2VyL2NvbXBvc2UvcmVsZWFzZXMvZG93bmxvYWQvMS4yOS4yL2RvY2tlci1jb21wb3NlLSQodW5hbWUgLXMpLSQodW5hbWUgLW0pIiAtbyAvdXNyL2xvY2FsL2Jpbi9kb2NrZXItY29tcG9zZQogIC0gc3VkbyBjaG1vZCAreCAvdXNyL2xvY2FsL2Jpbi9kb2NrZXItY29tcG9zZQogICMgaW5zdGFsbCBrdWJlY3RsCiAgLSBjdXJsIC1MTyAiaHR0cHM6Ly9kbC5rOHMuaW8vcmVsZWFzZS8kKGN1cmwgLUxTIGh0dHBzOi8vZGwuazhzLmlvL3JlbGVhc2Uvc3RhYmxlLnR4dCkvYmluL2xpbnV4L2FtZDY0L2t1YmVjdGwiCiAgLSBjaG1vZCAreCAuL2t1YmVjdGwKICAtIHN1ZG8gbXYgLi9rdWJlY3RsIC91c3IvbG9jYWwvYmluL2t1YmVjdGwK"
        }
      + platformConfig                : output<string>
      + preemptibleInstanceConfig     : output<string>
      + preserveBootVolume            : output<string>
      + privateIp                     : output<string>
      + publicIp                      : output<string>
      + region                        : output<string>
      + shape                         : "VM.Standard.E4.Flex"
      + shapeConfig                   : output<string>
      + sourceDetails                 : output<string>
      + state                         : output<string>
      + subnetId                      : output<string>
      + systemTags                    : output<string>
      + timeCreated                   : output<string>
      + timeMaintenanceRebootDue      : output<string>
      + updateOperationConstraint     : output<string>
      + urn                           : "urn:pulumi:prod::work-vm::oci:Core/instance:Instance::shukawam-compute"
    }

Resources:
    + 1 to create
    11 unchanged

Do you want to perform this update? yes
Updating (prod)

View in Browser (Ctrl+O): https://app.pulumi.com/shukawam/work-vm/prod/updates/32

     Type                  Name              Status
     pulumi:pulumi:Stack   work-vm-prod
 +   └─ oci:Core:Instance  shukawam-compute  created (45s)

Outputs:
  + instance: {
      + agentConfig              : {
          + areAllPluginsDisabled: false
          + isManagementDisabled : false
          + isMonitoringDisabled : false
          + pluginsConfigs       : []
        }
      + availabilityConfig       : {
          + isLiveMigrationPreferred: false
          + recoveryAction          : "RESTORE_INSTANCE"
        }
      + availabilityDomain       : "TGjA:AP-TOKYO-1-AD-1"
      + bootVolumeId             : "ocid1.bootvolume.oc1.ap-tokyo-1.abxhiljr4fq76fd367p7walkgevoxbcvek6bpd5zbkh6t7ibdfh3cmk6ntaa"
      + compartmentId            : "ocid1.compartment.oc1..."
      + createVnicDetails        : {
          + assignPrivateDnsRecord: false
          + assignPublicIp        : "true"
          + definedTags           : {}
          + displayName           : "shukawam"
          + freeformTags          : {}
          + hostnameLabel         : ""
          + nsgIds                : []
          + privateIp             : "192.168.0.193"
          + skipSourceDestCheck   : false
          + subnetId              : "ocid1.subnet.oc1.ap-tokyo-1.aaaaaaaavspfqma6sipvmkdlhwau4c4x7bubmkeqmmz6e7srj2htwzbcep5q"
          + vlanId                : ""
        }
      + definedTags              : {}
      + displayName              : "shukawam"
      + extendedMetadata         : {}
      + faultDomain              : "FAULT-DOMAIN-2"
      + freeformTags             : {}
      + hostnameLabel            : ""
      + id                       : "ocid1.instance.oc1.ap-tokyo-1.anxhiljrssl65iqcpuz2b3lqmkzqkusv6eq7544pebk6x3dzomvtpbq7o2za"
      + image                    : "ocid1.image.oc1.ap-tokyo-1.aaaaaaaai5l7ndbho5nvsql2fkkocbyaffdzzwfoaofgjvcrk2v6vpie5mtq"
      + instanceOptions          : {
          + areLegacyImdsEndpointsDisabled: false
        }
      + launchMode               : "PARAVIRTUALIZED"
      + launchOptions            : {
          + bootVolumeType                 : "PARAVIRTUALIZED"
          + firmware                       : "UEFI_64"
          + isConsistentVolumeNamingEnabled: true
          + isPvEncryptionInTransitEnabled : false
          + networkType                    : "PARAVIRTUALIZED"
          + remoteDataVolumeType           : "PARAVIRTUALIZED"
        }
      + metadata                 : {
          + user_data: "I2Nsb3VkLWNvbmZpZwpjaHBhc3N3ZDoKICBsaXN0OiB8CiAgICBzaHVrYXdhbTpDaGFuZ2VNZSEhCiAgZXhwaXJlOiBmYWxzZQp1c2VyczoKICAtIGRlZmF1bHQKICAtIG5hbWU6IHNodWthd2FtCiAgICBsb2NrX3Bhc3N3ZDogZmFsc2UKICAgIGdyb3Vwczogc3VkbywgdXNlcnMsIGFkbWluCiAgICBzaGVsbDogL2Jpbi9iYXNoCiAgICBzdWRvOiBbIkFMTD0oQUxMKSBOT1BBU1NXRD1BTEwiXQogICAgc3NoX2ltcG9ydF9pZDoKICAgICAgLSBnaDpzaHVrYXdhbQpzeXN0ZW1faW5mbzoKICBkZWZhdWx0X3VzZXI6CiAgICBuYW1lOiBkZWZhdWx0LXVzZXIKICAgIGxvY2tfcGFzc3dkOiBmYWxzZQogICAgc3VkbzogWyJBTEw9KEFMTCkgTk9QQVNTV0Q6QUxMIl0Kc3NoX3B3YXV0aDogbm8KcmFuZG9tX3NlZWQ6CiAgZmlsZTogL2Rldi91cmFuZG9tCiAgY29tbWFuZDogWyJwb2xsaW5hdGUiLCAiLXIiLCAiLXMiLCAiaHR0cHM6Ly9lbnRyb3B5LnVidW50dS5jb20iXQogIGNvbW1hbmRfcmVxdWlyZWQ6IHRydWUKcGFja2FnZV91cGdyYWRlOiB0cnVlCnBhY2thZ2VzOgogIC0gY3VybAogIC0gdmltCiAgLSBnaXQKICAtIHVuemlwCiAgLSBnbnVwZwogIC0gbHNiLXJlbGVhc2UKICAtIGNhLWNlcnRpZmljYXRlcwogIC0gZHN0YXQKcnVuY21kOgogICMgaW5zdGFsbCBkb2NrZXIKICAtIGN1cmwgLWZzU0wgaHR0cHM6Ly9kb3dubG9hZC5kb2NrZXIuY29tL2xpbnV4L3VidW50dS9ncGcgfCBzdWRvIGdwZyAtLWRlYXJtb3IgLW8gL3Vzci9zaGFyZS9rZXlyaW5ncy9kb2NrZXItYXJjaGl2ZS1rZXlyaW5nLmdwZwogIC0gZWNobyAiZGViIFthcmNoPSQoZHBrZyAtLXByaW50LWFyY2hpdGVjdHVyZSkgc2lnbmVkLWJ5PS91c3Ivc2hhcmUva2V5cmluZ3MvZG9ja2VyLWFyY2hpdmUta2V5cmluZy5ncGddIGh0dHBzOi8vZG93bmxvYWQuZG9ja2VyLmNvbS9saW51eC91YnVudHUgJChsc2JfcmVsZWFzZSAtY3MpIHN0YWJsZSIgfCBzdWRvIHRlZSAvZXRjL2FwdC9zb3VyY2VzLmxpc3QuZC9kb2NrZXIubGlzdCA+IC9kZXYvbnVsbAogIC0gc3VkbyBhcHQtZ2V0IHVwZGF0ZQogIC0gc3VkbyBhcHQtZ2V0IGluc3RhbGwgLXkgZG9ja2VyLWNlIGRvY2tlci1jZS1jbGkgY29udGFpbmVyZC5pbwogIC0gc3VkbyBjdXJsIC1MICJodHRwczovL2dpdGh1Yi5jb20vZG9ja2VyL2NvbXBvc2UvcmVsZWFzZXMvZG93bmxvYWQvMS4yOS4yL2RvY2tlci1jb21wb3NlLSQodW5hbWUgLXMpLSQodW5hbWUgLW0pIiAtbyAvdXNyL2xvY2FsL2Jpbi9kb2NrZXItY29tcG9zZQogIC0gc3VkbyBjaG1vZCAreCAvdXNyL2xvY2FsL2Jpbi9kb2NrZXItY29tcG9zZQogICMgaW5zdGFsbCBrdWJlY3RsCiAgLSBjdXJsIC1MTyAiaHR0cHM6Ly9kbC5rOHMuaW8vcmVsZWFzZS8kKGN1cmwgLUxTIGh0dHBzOi8vZGwuazhzLmlvL3JlbGVhc2Uvc3RhYmxlLnR4dCkvYmluL2xpbnV4L2FtZDY0L2t1YmVjdGwiCiAgLSBjaG1vZCAreCAuL2t1YmVjdGwKICAtIHN1ZG8gbXYgLi9rdWJlY3RsIC91c3IvbG9jYWwvYmluL2t1YmVjdGwK"
        }
      + platformConfig           : <null>
      + preemptibleInstanceConfig: <null>
      + privateIp                : "192.168.0.193"
      + publicIp                 : "155.248.166.94"
      + region                   : "ap-tokyo-1"
      + shape                    : "VM.Standard.E4.Flex"
      + shapeConfig              : {
          + baselineOcpuUtilization  : ""
          + gpuDescription           : ""
          + gpus                     : 0
          + localDiskDescription     : ""
          + localDisks               : 0
          + localDisksTotalSizeInGbs : 0
          + maxVnicAttachments       : 2
          + memoryInGbs              : 16
          + networkingBandwidthInGbps: 1
          + nvmes                    : 0
          + ocpus                    : 1
          + processorDescription     : "2.55 GHz AMD EPYC™ 7J13 (Milan)"
        }
      + sourceDetails            : {
          + bootVolumeSizeInGbs: "47"
          + bootVolumeVpusPerGb: "10"
          + kmsKeyId           : ""
          + sourceId           : "ocid1.image.oc1.ap-tokyo-1.aaaaaaaai5l7ndbho5nvsql2fkkocbyaffdzzwfoaofgjvcrk2v6vpie5mtq"
          + sourceType         : "image"
        }
      + state                    : "RUNNING"
      + subnetId                 : "ocid1.subnet.oc1.ap-tokyo-1.aaaaaaaavspfqma6sipvmkdlhwau4c4x7bubmkeqmmz6e7srj2htwzbcep5q"
      + systemTags               : {}
      + timeCreated              : "2023-12-01 06:13:14.597 +0000 UTC"
      + timeMaintenanceRebootDue : ""
      + urn                      : "urn:pulumi:prod::work-vm::oci:Core/instance:Instance::shukawam-compute"
    }

Resources:
    + 1 created
    11 unchanged

Duration: 47s
```

※実行結果は一部マスクしています

Outputs に含まれている publicIp に SSH 接続してみます。

```sh
ssh shukawam@155.248.166.94

The authenticity of host '155.248.166.94 (155.248.166.94)' can't be established.
ED25519 key fingerprint is SHA256:38ROfauuCoIpND+MNk3R0aJjpRjq+KZq8fi4kj9P/oY.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '155.248.166.94' (ED25519) to the list of known hosts.
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-1045-oracle x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Dec  1 06:38:08 UTC 2023

  System load:  0.0               Processes:                114
  Usage of /:   8.1% of 44.96GB   Users logged in:          0
  Memory usage: 2%                IPv4 address for docker0: 172.17.0.1
  Swap usage:   0%                IPv4 address for ens3:    192.168.0.193

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


*** System restart required ***
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

shukawam@shukawam:~$
```

Docker, kubectl がインストールされていることも確認できます。

```sh
docker --version
Docker version 24.0.7, build afdd53b

kubectl version
Client Version: v1.28.4
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

# 終わりに

Pulumi を使って OCI のインフラ構築を効率化してみました。これに限らず、様々なリソースに対応しているので是非色々と試してみてください！

今回作成したコードは GitHub に格納していますので、詳細な部分が気になった方はこちらも併せてご参照ください。

https://github.com/shukawam/pulumi-stacks/tree/main/oci/work-vm

