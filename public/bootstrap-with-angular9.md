---
title: Angular9でBootstrap4を使う
tags:
  - Node.js
  - Bootstrap
  - Angular
private: false
updated_at: '2021-08-26T08:29:34+09:00'
id: 1134147d5ac61789987d
organization_url_name: null
slide: false
ignorePublish: false
---
# 始めに

- Angular 9系にBootstrap4 (ng-bootstrap) を適用する手順です。
- 通常の Bootstrap（jQuery, popper.js依存）を使用してもよいですが、余計なライブラリに依存することになる事になるため、おすすめしません。
  - ng-bootstrapは、Bootstrapが依存しているjQuery, popper.jsの実装をAngularのcomponentに差し替えています。
- メジャーバージョンはしっかりと確認する。
  - 特に Angular5 と Angular6+では CLI の設定ファイル周りが大きく変更となっています。
- 公式の英語ドキュメント読むのめんどいって方向け。

# 環境

タイトルにもある通り、今回はAngular9系にBootstrap4を適用します。

```
$ ng --version 
     _                      _                 ____ _     ___
    / \   _ __   __ _ _   _| | __ _ _ __     / ___| |   |_ _|
   / △ \ | '_ \ / _` | | | | |/ _` | '__|   | |   | |    | |
  / ___ \| | | | (_| | |_| | | (_| | |      | |___| |___ | |
 /_/   \_\_| |_|\__, |\__,_|_|\__,_|_|       \____|_____|___|
                |___/
    

Angular CLI: 9.0.7
Node: 12.14.1
OS: linux x64

Angular: 9.0.7
... animations, cli, common, compiler, compiler-cli, core, forms
... language-service, platform-browser, platform-browser-dynamic
... router
Ivy Workspace: Yes

Package                           Version
-----------------------------------------------------------
@angular-devkit/architect         0.900.7
@angular-devkit/build-angular     0.900.7
@angular-devkit/build-optimizer   0.900.7
@angular-devkit/build-webpack     0.900.7
@angular-devkit/core              9.0.7
@angular-devkit/schematics        9.0.7
@angular/localize                 9.1.1
@ngtools/webpack                  9.0.7
@schematics/angular               9.0.7
@schematics/update                0.900.7
rxjs                              6.5.5
typescript                        3.7.5
webpack                           4.41.2
```



# 手順

## 2021/08/26 追記

今でも割と見られているようなので追記します。Angular 9+ の場合は、ワンライナーで適用できるようになったみたいです。

```bash
ng add @ng-bootstrap/ng-bootstrap
```

## 手動でインストールする場合

Angular 9 未満の場合は、CLI が対応していないので以下の手順を踏む必要があります。

### ng-bootstrapをインストールする

Angular 9.0.0 以上かつ ng-bootstrap 6.0.0 以上の場合は`@angular/localize`をpolyfillsに追加する必要があります。

```bash
ng add @angular/localize
```



`ng-bootstrap`をインストールする。（Bootstrapに依存しているため、一緒にインストールします。）

```bash
npm install @ng-bootstrap/ng-bootstrap bootstrap
```



### ルートモジュールにインポートする。

```app.module.ts
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';

import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app.component';
import { NgbModule } from '@ng-bootstrap/ng-bootstrap';

@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule,
    AppRoutingModule,
    NgbModule,
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```



### angular.jsonに設定を追加する

グローバルスタイルとして、bootstrap を使用するために以下を追記する。

```angular.json
"styles": [
  "./node_modules/bootstrap/dist/css/bootstrap.min.css",
  "src/styles.css"
]
```



## 動作確認

```app.component.html
<button type="button" class="btn btn-primary">test</button>
```



アプリケーションを起動後、[http://localhost:4200](http://localhost:4200)にアクセスし、以下の画面が見えればOKです。

![image-20200412180154202.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/84643041-df04-a3b6-8330-5fbe999b452c.png)



# 参考

- [ng-bootstrap#getting-started](https://ng-bootstrap.github.io/#/getting-started)
