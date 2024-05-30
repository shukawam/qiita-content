---
title: Grafana の SSO 用の Keycloak を Keycloak Provider から設定してみた
tags:
  - grafana
  - Terraform
  - Keycloak
private: false
updated_at: '2023-06-15T16:16:11+09:00'
id: 13cc65a06fa159841276
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

以前、Terraform - Keycloak Provider の Getting Started 的な記事を書きました。

https://qiita.com/shukawam/items/9fe125b732af125ff402

本記事では、上記の応用編として Keycloak の Realm, Group, User, Client を Keycloak Provider を用いて作成し、Grafana に Keycloak を用いて SSO できるところまでを確認します。例によって、Terraform、Keycloak の基本的な説明は割愛します。

# Grafana の要件を確認する

Keycloak として設定する必要のある項目は、この辺りが参考になります。

https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/generic-oauth/

今回は、

- Keycloak 上の admin グループに属するユーザーを Grafana の Admin ロールに割り当て
- Keycloak 上の guest グループに属するユーザーを Grafana の Viewer ロールに割り当て

としたいので、発行される ID Token の中に groups という Claim を含むように Keycloak を設定します。

# Keycloak の構成ファイル

今回は、上記要件を満たすために以下のような Terraform の構成ファイルを作ってみました。

```hcl:main.tf
#####
# Realm
resource "keycloak_realm" "realm" {
  realm = local.realm_id
}

#####
# Groups
resource "keycloak_group" "admin" {
  name     = "admin"
  realm_id = keycloak_realm.realm.id
}

resource "keycloak_group" "guest" {
  name     = "guest"
  realm_id = keycloak_realm.realm.id
}

#####
# Users
resource "keycloak_user" "admin_users" {
  for_each       = { for i in var.admin_users : i.username => i }
  realm_id       = keycloak_realm.realm.id
  username       = each.value.username
  enabled        = true
  email          = each.value.email
  email_verified = true
  first_name     = each.value.first_name
  last_name      = each.value.last_name
  initial_password {
    value = local.initial_password
  }
}

resource "keycloak_user" "guest_users" {
  for_each       = { for i in var.guest_users : i.username => i }
  realm_id       = keycloak_realm.realm.id
  username       = each.value.username
  enabled        = true
  email          = each.value.email
  email_verified = true
  first_name     = each.value.first_name
  last_name      = each.value.last_name
  initial_password {
    value = local.initial_password
  }
}

#####
# Group membership
resource "keycloak_group_memberships" "admin_group_membershop" {
  for_each = { for i in var.admin_users : i.username => i }
  realm_id = keycloak_realm.realm.id
  group_id = keycloak_group.admin.id
  members = [
    each.value.username
  ]
}

resource "keycloak_group_memberships" "guest_group_membershop" {
  for_each = { for i in var.guest_users : i.username => i }
  realm_id = keycloak_realm.realm.id
  group_id = keycloak_group.guest.id
  members = [
    each.value.username
  ]
}

#####
# OpenID Connect Client Scope
resource "keycloak_openid_client_scope" "groups" {
  realm_id               = keycloak_realm.realm.id
  name                   = local.scope_name_groups
  include_in_token_scope = true
}

resource "keycloak_openid_group_membership_protocol_mapper" "groups_mapper" {
  realm_id        = keycloak_realm.realm.id
  client_scope_id = keycloak_openid_client_scope.groups.id
  name            = local.scope_name_groups
  claim_name      = local.scope_name_groups
  full_path       = false
}

#####
# OpenID Connect Client
resource "keycloak_openid_client" "grafana_client" {
  realm_id              = keycloak_realm.realm.id
  enabled               = true
  name                  = local.client_name
  client_id             = local.client_id
  access_type           = "CONFIDENTIAL"
  standard_flow_enabled = true
  valid_redirect_uris   = var.valid_redirect_uris
}

resource "keycloak_openid_client_default_scopes" "grafana_client_default_scopes" {
  realm_id       = keycloak_realm.realm.id
  client_id      = keycloak_openid_client.grafana_client.id
  default_scopes = local.default_scopes
}
```

簡単に解説をします。まずは、以下の部分

```hcl
#####
# Groups
resource "keycloak_group" "admin" {
  name     = "admin"
  realm_id = keycloak_realm.realm.id
}

resource "keycloak_group" "guest" {
  name     = "guest"
  realm_id = keycloak_realm.realm.id
}
```

admin(Keycloak) -> Admin(Grafana), guest(Keycloak) -> Viewer(Grafana)に割り当てるための Keycloak のグループを作成しています。

次に、この部分

```hcl
#####
# Users
resource "keycloak_user" "admin_users" {
  for_each       = { for i in var.admin_users : i.username => i }
  realm_id       = keycloak_realm.realm.id
  username       = each.value.username
  enabled        = true
  email          = each.value.email
  email_verified = true
  first_name     = each.value.first_name
  last_name      = each.value.last_name
  initial_password {
    value = local.initial_password
  }
}

resource "keycloak_user" "guest_users" {
  for_each       = { for i in var.guest_users : i.username => i }
  realm_id       = keycloak_realm.realm.id
  username       = each.value.username
  enabled        = true
  email          = each.value.email
  email_verified = true
  first_name     = each.value.first_name
  last_name      = each.value.last_name
  initial_password {
    value = local.initial_password
  }
}
```

`variables.tf(vars)`, `locals.tf` に定義している値を用いて admin/guest グループに属するユーザーを作成しています。ユーザーは以下のように定義します。

```variable.tf
variable "admin_users" {
  description = "Member of admin group"
}

variable "guest_users" {
  description = "Member of guest group"
}
```

```variables.tfvars
admin_users = [
  {
    "username"   = "admin",
    "email"      = "admin@example.com",
    "first_name" = "Hoge",
    "last_name"  = "Fuga"
  }
]
guest_users = [
  {
    "username"   = "guest",
    "email"      = "guest@example.com",
    "first_name" = "Hoge",
    "last_name"  = "Fuga"
  }
]
```

パスワードは、初期パスワードを設定し Realm 作成後に入れ替えるようにしています。

```locals.tf
locals {
  initial_password = "ChangeMe!!"
}
```

以下部分では、作成した admin/guest グループとユーザーの紐づけを実施しています。

```hcl
#####
# Group membership
resource "keycloak_group_memberships" "admin_group_membershop" {
  for_each = { for i in var.admin_users : i.username => i }
  realm_id = keycloak_realm.realm.id
  group_id = keycloak_group.admin.id
  members = [
    each.value.username
  ]
}

resource "keycloak_group_memberships" "guest_group_membershop" {
  for_each = { for i in var.guest_users : i.username => i }
  realm_id = keycloak_realm.realm.id
  group_id = keycloak_group.guest.id
  members = [
    each.value.username
  ]
}
```

以下部分では、ID Token に含める groups claim の定義とその Claim に対する mapper を定義します。

```hcl
#####
# OpenID Connect Client Scope
resource "keycloak_openid_client_scope" "groups" {
  realm_id               = keycloak_realm.realm.id
  name                   = local.scope_name_groups
  include_in_token_scope = true
}

resource "keycloak_openid_group_membership_protocol_mapper" "groups_mapper" {
  realm_id        = keycloak_realm.realm.id
  client_scope_id = keycloak_openid_client_scope.groups.id
  name            = local.scope_name_groups
  claim_name      = local.scope_name_groups
  full_path       = false
}
```

最後に、Grafana 用の OpenID Connect Client とそこに含めるデフォルトの scope を定義しています。

```hcl
#####
# OpenID Connect Client
resource "keycloak_openid_client" "grafana_client" {
  realm_id              = keycloak_realm.realm.id
  enabled               = true
  name                  = local.client_name
  client_id             = local.client_id
  access_type           = "CONFIDENTIAL"
  standard_flow_enabled = true
  valid_redirect_uris   = var.valid_redirect_uris
}

resource "keycloak_openid_client_default_scopes" "grafana_client_default_scopes" {
  realm_id       = keycloak_realm.realm.id
  client_id      = keycloak_openid_client.grafana_client.id
  default_scopes = local.default_scopes
}
```

# Grafana の設定

個人的な環境では、kube-prometheus-stack を用いて Grafana を構築しているので、その環境に合わせた設定ファイルとなっていることをご了承ください。（良しなに、自身のディストリビューションや構築しているインフラに合わせて読み替えてください）

```yaml
grafana.ini:
  server:
    root_url: https://<grafana>
  auth.generic_oauth:
    enabled: true
    name: Keycloak
    allow_sign_up: true
    scopes: openid profile groups email
    auth_url: https://<keycloak>/realms/<realm>/protocol/openid-connect/auth
    token_url: https://<keycloak>/realms/<realm>/protocol/openid-connect/token
    api_url: https://<keycloak>/realms/<realm>/protocol/openid-connect/userinfo
    role_attribute_path: contains(groups[*], 'admin') && 'Admin' || contains(groups[*], 'guest') && 'Editor' || 'Viewer'
  envValueFrom:
    GF_AUTH_GENERIC_OAUTH_CLIENT_ID:
      secretKeyRef:
      name: grafana-keycloak-secret
      key: client_id
    GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET:
      secretKeyRef:
      name: grafana-keycloak-secret
      key: client_secret
```

`server.root_uri` や `auth.generic_oauth.auth_url` などに含まれる URL はご自身の環境に合わせて修正してください。`auth.generic_oauth` に、existingSecret とかそういうフィールドがあるのかと思っていたのですがどうやらなさそうで、右往左往していたのですが普通に環境変数から client_id/secret を渡せました。（`GF_AUTH_GENERIC_OAUTH_CLIENT_ID`, `GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET`）

# 確認

Keycloak でログインするためのオプションが出てきます。

![image06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/c3a383a0-2df6-bbd9-4a67-4a713af87585.png)

作成したユーザーでログインします。

![image07.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/1e6c6c51-9583-a480-19eb-788a5fe34499.png)

確認できました。

![image08.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/9650350a-363c-d8dc-73ed-54468ec88061.png)

# おわりに

variables.tf, locals.tf 辺りはかなり説明を割愛したので、気になる方はこちらのリポジトリをご参照ください。

https://github.com/shukawam/manifests

- [terraform/keycloak/\*](https://github.com/shukawam/manifests/tree/main/terraform/keycloak): 今回用いた Keycloak 設定用の Terraform の構成ファイル群
- [app/kube-prometheus-stack.yaml](https://github.com/shukawam/manifests/blob/main/app/kube-prometheus-stack.yaml): kube-prometheus-stack の ArgoCD - Application の定義
