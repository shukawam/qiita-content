---
title: Terraform の Keycloak Provider を試してみる
tags:
  - Terraform
  - Keycloak
private: false
updated_at: '2023-06-14T11:54:33+09:00'
id: 9fe125b732af125ff402
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

Terraform に [Keycloak Provider](https://registry.terraform.io/providers/mrparkers/keycloak/latest) が存在することを発見したので、これを試してみます。Keycloak では、Realm の情報をインポート／エクスポートする機能が存在しますが JSON でこれ扱うため可読性には若干かけます。Terraform - Keycloak Provider では HCL で宣言的に Realm の設定を定義できるとのことで個人的に素敵だと感じています。

本記事は、Keycloak が構築済みであることを前提に執筆します。また、Keycloak Provider に焦点を当てるため Terraform に関する基本的な事項や Keycloak に関する説明、途中にでてくる標準仕様（OpenID Connect, ...）に関する説明は割愛させていただきます。

## Terraform - Keycloak Provider の認証

Terraform - Keycloak Provider が Keycloak の設定をするために認証が必要です。これには、`admin-cli` クライアントを用いるか Terraform 用に OIDC(OpenID Connect)クライアントを作成する方法が存在しますが、

> Client Credentials Grant Setup (recommended)

と記載があり、OIDC クライアントを作成する方が推奨とのことなので、Terraform 用にクライアントを作成します。

今回は、Realm の作成から Terraform で実施するため、`master` realm で以下のようにクライアントを作成します。該当 Realm 内に関する操作のみで良い場合は、Realm 内に OIDC クライアントを作成してください。

**General Settings**

- Client type: OpenID Connect
- Client ID: terraform
- Name: Terraform
- Description: Terraform Keycloak Provider
- Always display in console: off

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/aceec0fc-1a43-6304-dc1c-8e1ca66b9341.png)

**Capability config**

- Client authentication: On
- Authorization: Off
- Authentication flow: Service accounts role にチェック

OpenID Connect でいうところの Client Credentials Flow を実現するための設定です。

![image02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/c2c832ec-387c-38b1-76b9-25a6f6b41943.png)

Save すると、Credentials Tab に Client Secret が生成されているので、これを控えておきます。

![image03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/eaf70d2a-c5cd-edc0-4d45-2af82be5a146.png)

次に、このクライアントに対して admin role を割り当てます。

![image04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/50cf811b-99fb-5c95-2d40-2a602515fc3e.png)

割り当てる Role を制限したい場合は、[Assigning Roles](https://registry.terraform.io/providers/mrparkers/keycloak/latest/docs#assigning-roles) を参考にして設定してください。

## Terraform 構成ファイルを書く

```hcl:provider.tf
provider "keycloak" {
  client_id     = var.client_id
  client_secret = var.client_secret
  url           = var.url
}

terraform {
  required_providers {
    keycloak = {
      source  = "mrparkers/keycloak"
      version = ">= 4.0.0"
    }
  }
}
```

```hcl:variables.tf
variable "client_id" {
  description = "Keycloak client_id for Terraform"
}

variable "client_secret" {
  description = "Keycloak client_secret for Terraform"
}

variable "url" {
  description = "Keycloak URL"
}
```

入力変数は環境変数からでも `*.tfvars` からでも何でも良いので適当に渡してください。

ここまでで、Keycloak Provider と Keycloak 間の認証は完了したので、動作確認のために Realm を作ってみましょう。

```hcl:main.tf
#####
# Realm
locals {
  realm_id = "example-realm" # ご自由な名前でどうぞ
}

resource "keycloak_realm" "realm" {
  realm = local.realm_id
}
```

init

```bash
$ terraform init

Initializing the backend...

Initializing provider plugins...
- Finding mrparkers/keycloak versions matching ">= 4.0.0"...
- Installing mrparkers/keycloak v4.3.1...
- Installed mrparkers/keycloak v4.3.1 (self-signed, key ID C50867915E116CD2)

Partner and community providers are signed by their developers.
If you'd like to know more about provider signing, you can read about it here:
https://www.terraform.io/docs/cli/plugins/signing.html

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

plan

```bash
$ terraform plan -var-file variables.tfvars

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # keycloak_realm.realm will be created
  + resource "keycloak_realm" "realm" {
      + access_code_lifespan                     = (known after apply)
      + access_code_lifespan_login               = (known after apply)
      + access_code_lifespan_user_action         = (known after apply)
      + access_token_lifespan                    = (known after apply)
      + access_token_lifespan_for_implicit_flow  = (known after apply)
      + action_token_generated_by_admin_lifespan = (known after apply)
      + action_token_generated_by_user_lifespan  = (known after apply)
      + browser_flow                             = (known after apply)
      + client_authentication_flow               = (known after apply)
      + client_session_idle_timeout              = (known after apply)
      + client_session_max_lifespan              = (known after apply)
      + direct_grant_flow                        = (known after apply)
      + docker_authentication_flow               = (known after apply)
      + duplicate_emails_allowed                 = (known after apply)
      + edit_username_allowed                    = (known after apply)
      + enabled                                  = true
      + id                                       = (known after apply)
      + internal_id                              = (known after apply)
      + login_with_email_allowed                 = (known after apply)
      + oauth2_device_code_lifespan              = (known after apply)
      + oauth2_device_polling_interval           = (known after apply)
      + offline_session_idle_timeout             = (known after apply)
      + offline_session_max_lifespan             = (known after apply)
      + offline_session_max_lifespan_enabled     = false
      + realm                                    = "example-realm"
      + refresh_token_max_reuse                  = 0
      + registration_allowed                     = (known after apply)
      + registration_email_as_username           = (known after apply)
      + registration_flow                        = (known after apply)
      + remember_me                              = (known after apply)
      + reset_credentials_flow                   = (known after apply)
      + reset_password_allowed                   = (known after apply)
      + revoke_refresh_token                     = false
      + ssl_required                             = "external"
      + sso_session_idle_timeout                 = (known after apply)
      + sso_session_idle_timeout_remember_me     = (known after apply)
      + sso_session_max_lifespan                 = (known after apply)
      + sso_session_max_lifespan_remember_me     = (known after apply)
      + user_managed_access                      = false
      + verify_email                             = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.
```

apply

```bash
$ terraform apply -var-file variables.tfvars

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # keycloak_realm.realm will be created
  + resource "keycloak_realm" "realm" {
      + access_code_lifespan                     = (known after apply)
      + access_code_lifespan_login               = (known after apply)
      + access_code_lifespan_user_action         = (known after apply)
      + access_token_lifespan                    = (known after apply)
      + access_token_lifespan_for_implicit_flow  = (known after apply)
      + action_token_generated_by_admin_lifespan = (known after apply)
      + action_token_generated_by_user_lifespan  = (known after apply)
      + browser_flow                             = (known after apply)
      + client_authentication_flow               = (known after apply)
      + client_session_idle_timeout              = (known after apply)
      + client_session_max_lifespan              = (known after apply)
      + direct_grant_flow                        = (known after apply)
      + docker_authentication_flow               = (known after apply)
      + duplicate_emails_allowed                 = (known after apply)
      + edit_username_allowed                    = (known after apply)
      + enabled                                  = true
      + id                                       = (known after apply)
      + internal_id                              = (known after apply)
      + login_with_email_allowed                 = (known after apply)
      + oauth2_device_code_lifespan              = (known after apply)
      + oauth2_device_polling_interval           = (known after apply)
      + offline_session_idle_timeout             = (known after apply)
      + offline_session_max_lifespan             = (known after apply)
      + offline_session_max_lifespan_enabled     = false
      + realm                                    = "example-realm"
      + refresh_token_max_reuse                  = 0
      + registration_allowed                     = (known after apply)
      + registration_email_as_username           = (known after apply)
      + registration_flow                        = (known after apply)
      + remember_me                              = (known after apply)
      + reset_credentials_flow                   = (known after apply)
      + reset_password_allowed                   = (known after apply)
      + revoke_refresh_token                     = false
      + ssl_required                             = "external"
      + sso_session_idle_timeout                 = (known after apply)
      + sso_session_idle_timeout_remember_me     = (known after apply)
      + sso_session_max_lifespan                 = (known after apply)
      + sso_session_max_lifespan_remember_me     = (known after apply)
      + user_managed_access                      = false
      + verify_email                             = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

keycloak_realm.realm: Creating...
keycloak_realm.realm: Creation complete after 5s [id=example-realm]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

Realm の一覧を確認してみると、example-realm が含まれていることが確認できます。

![image05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/b91364cd-01fa-04ad-4ae3-d7b81519371b.png)

# 終わりに

今回は、Terraform Keycloak Provider を使って Keycloak の設定を IaC 化してみました。

https://registry.terraform.io/providers/mrparkers/keycloak/latest/docs

を確認してみると、数多くの Resource が提供されていることが確認できます。次回は、この Keycloak Provider を使って Grafana の SSO 用の Realm, Group, Member, Client などを作ってみたいと思います。
