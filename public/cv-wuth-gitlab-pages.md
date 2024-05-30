---
title: 職務経歴書をGitLab Pages を使って公開してみた
tags:
  - GitLab
  - gitbook
  - GitLab-CI
  - gitlab-pages
private: false
updated_at: '2021-04-02T13:28:51+09:00'
id: 955da1b76387e5b3a237
organization_url_name: null
slide: false
ignorePublish: false
---
# 始めに

長期の休みとなり、まとまった時間が取れたので自分の２年間の振り返り、いざという時のために職務経歴書なるものを書いてみようと思った。フォーマットを調べてみると、PDF や Word といった形式が主流らしい。最初は、Wordで書いてPDFに変換しようと考えていたが、まがいなりにも私はエンジニアなので、~~Wordのレイアウトと格闘するなどの~~余計なコストは使わずに、Markdownでさくっと書いて構成管理をしたいと思った。そこで個人的に使用している GitLab で構成管理、特に隠す理由もないので GitLab Pages で公開することにした。



# 作業内容

実現したいことは以下の通り。

- 職務経歴書（Markdown）を構成管理したい。
- MasterにマージしたタイミングでPDFに変換しダウンロードできるようにしたい。
- 別に隠す必要もないので GitLab Pages として公開する。



## リポジトリ作成

特に名前に決まりはないが、英約にすると **Curriculum Vitae** ということだったので、`curriculum-vitae`という名前にした。
以下、リポジトリのディレクトリ構成です。（GitBookのディレクトリ構成に倣っています）

```
curriculum-vitae
|- styles
|   |- pdf.css
|   |- website.css
|- gitlab-ci.yml
|- book.json
|- README.md
|- SUMMARY.md
```

- styles: 公開サイトやPDFに適用するスタイルシート
- .gitlab-ci.yml: GitLab CI/CD で実行されるスクリプトを定義
- book.json: GitBookの設定ファイル

  以下、設定例です。

  ```book.json
  {
    "langeage": "ja",
    "title": "Curriculum Vitae",
    "styles": {
      "website": "styles/website.css",
      "pdf": "styles/pdf.css"
    },
    "plugins": ["theme-api", "-sharing"]
  }
  ```

- README.md: 職務経歴書の本体
- SUMMARY.md: 文章の構成



## 職務経歴書を書く

各々の経歴をご自由に記載してください。何を書くとかは決まってないですが、私はこの辺りを参考に書きました。

- [エンジニアが読みたくなる職務経歴書](https://dwango.github.io/articles/engineers-resume/)


## GitLab Pagesで公開する準備

### .gitlab-ci.ymlを書く

GitLab CI/CD を使って、静的サイトのビルド -> 公開、MarkdownからPDFへの変換を行います。
以下のファイルをリポジトリのルートに配置する。

```gitlab-ci.yml
image: node:10

cache:
  paths:
    - node_modules/

before_script:
  - apt-get update
  - apt-get install -y calibre xvfb fonts-ipafont-gothic fonts-ipafont-mincho
  - npm install -g gitbook-cli
  - gitbook fetch 3.2.3
  - gitbook install

test:
  stage: test
  script:
    - gitbook build . public
    - xvfb-run gitbook pdf
  only:
    - branches
  except:
    - master

pages:
  stage: deploy
  script:
    - gitbook build . public
    - xvfb-run gitbook pdf . curriculum-vitae.pdf
  artifacts:
    paths:
      - public
      - curriculum-vitae.pdf
    expire_in: 1 week
  only:
    - master
```



### `git push`する

無事成功すると、GitLab Pages として先ほど記載した職務経歴書が公開され、提出用の PDF が artifacts として出力されます。

![image-20200427152223532.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/6ee6a0da-c426-ef70-6ca7-551c84c1160d.png)


試しに、ダウンロードしてみる。

![image001.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/fe6bcea6-33eb-d26e-7510-7062174098fb.png)


zipで圧縮されているので、適当に解凍します。

![image-20200427153753045.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/a887040a-32e9-c3eb-9b4c-6ec3df5865a5.png)


- public；GitLab Pagesとして公開されている資産一式。
- curriculum-vitae.pdf；Markdownで記述した職務経歴書をPDFに変換したもの。


GitLab Pagesは、特に設定を加えなければ、**https://<ユーザ名>.gitlab.io/curriculum-vitae**で公開されます。
※ユーザー名に`.`が含まれていると証明書関係のエラーになりますがコンテンツ的には問題ないので大丈夫です。

# 終わりに

思い付きでやってみましたが、自分の経験・スキルを振り返るのはいい取り組みだなと思いました。半年に一回くらいの頻度でアップデートしたい。また、今回の仕組みに関してもまだまだ改善ポイントがありそうです。（特にGitBookの設定周り）



# 参考

- [職務経歴書をGitHubで管理しよう](https://qiita.com/okohs/items/abcad0b4aefa585bc50b)

- [GitBookによるドキュメント作成](https://qiita.com/mebiusbox2/items/938af4b0d0bf7a4d3e33)

- [gitbookで楽々ドキュメント作成](https://www.cresco.co.jp/blog/entry/2509/)
