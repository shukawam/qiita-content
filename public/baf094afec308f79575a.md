---
title: WSL2でWindows10の開発環境を整える
tags:
  - Ubuntu
  - Docker
  - Windows10
  - WSL
  - WSL2
private: false
updated_at: '2021-03-29T21:20:35+09:00'
id: baf094afec308f79575a
organization_url_name: null
slide: false
ignorePublish: false
---
# 作成する環境

- Windows Subsystem for Linux(WSL)で、Docker 環境を作成する。

# 環境

- Windows 10 Home
  - バージョン：1903

# 作業ログ

## WSL を有効にする

以下を参考に WSL 環境を構築する。

[https://www.atmarkit.co.jp/ait/articles/1608/08/news039.html](https://www.atmarkit.co.jp/ait/articles/1608/08/news039.html)

## Ubuntu を導入する

Microsoft Store から Ubuntu をインストールします。（Ubuntu 18.04 LTS）

※2020/01/02 現在では最新の LTS
![01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/08862144-47d0-9624-f5f2-866fd055ae77.png)

インストール作業終了後、Ubuntu を管理者権限で起動します。

## 各種初期設定

とりあえずここまでで、Ubuntu 環境が作成できました。

※以下は、自分が初期セットアップに実施したこと。

### リポジトリの変更

データの取得先を海外のサーバから日本のサーバへと変更する。

```bash
sudo sed -i -e 's%http://.*.ubuntu.com%http://ftp.jaist.ac.jp/pub/Linux%g' /etc/apt/sources.list
```

### パッケージのアップデート

```bash
sudo apt update
sudo apt upgrade
```

### DNS サーバを変更

※2020/01/10 追記

`git clone`しようとしたときに**gitlab.com**が名前解決できなかったため。

**/etc/systemd/resolved.conf**に以下の項目を追記

```bash
[Resolve]
DNS=8.8.8.8
```

## Docker の環境構築

※WSL 上で Docker の動作が確認されているのは、**17.x**までとなります。

**18.x**はまだサポートされていないと思われます。

### docker 17.12.1 のインストール

```bash
sudo apt install docker.io=17.12.1-0ubuntu1
sudo cgroupfs-mount
# 現在のユーザでdockerコマンドをsudoせずに実行できるようにする。
sudo usermod -aG docker $USER
```

### Docker デーモンの起動

```bash
sudo service docker start
```

※2020/01/10 追記

↑ この方法ではだめでした。

[WSL の docker daemon を自動起動させる](https://qiita.com/forest1/items/ab6d8b345653c614229b)を参考に再度設定を行う。

以下のシェルスクリプトを作成（置き場は任意）

```shell
#!/bin/sh
sudo cgroupfs-mount
sudo usermod -aG docker $USER
sudo service docker start
```

タスクスケジューラーに登録

（Windows スタートメニュ -> Windows 管理ツール -> タスクスケジューラ）

![02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/e51984f9-56fb-2187-e668-992ea3545031.png)

> 右側の「タスクの作成」を選択して以下を設定します。
>
> 全般
> 「名前」：Docker (任意で)
> 「最上位の特権で実行する」にチェックを入れる（必須！）
>
> トリガー
> 「新規」で作成し、
> 「タスクの開始」：ログオン時を選択
> 「設定」：特定のユーザを選択
>
> 操作
> 「新規」で作成し、
> 「操作」：プログラムの開始を選択
> 「プログラム/スクリプト」：wsl.exe
> 「引数の追加」：<作成したスクリプトのパスを指定>
> 　パスは WSL(Linux)形式で指定します。(C:ドライブ以下は、/mnt/c/)
>
> 条件
> 「コンピューターを AC 電源で使用している場合のみタスクを開始する」チェックを外す
>
> 設定
> 「タスクが失敗した場合の再起動の間隔」：1 分間
>
> 以上が完了したら、PC を再起動させます。

再起動後に`sudo`のパスワードが求められるが、個人利用の PC のためパスワード入力を省略する。

```bash
sudo visudo
```

```bash
#
# This file MUST be edited with the 'visudo' command as root.
#
# Please consider adding local content in /etc/sudoers.d/ instead of
# directly modifying this file.
#
# See the man page for details on how to write a sudoers file.
#
Defaults        env_reset
Defaults        mail_badpass
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"

# Host alias specification

# User alias specification

# Cmnd alias specification

# User privilege specification
root    ALL=(ALL:ALL) ALL

# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL

# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL

# Allow members to execute sudo command without password <- 追記
<自分のユーザ名> ALL=NOPASSWD: ALL <- 追記

# See sudoers(5) for more information on "#include" directives:

#includedir /etc/sudoers.d
```

### 起動時に自動的に Docker デーモンが起動されるようにする

```bash
sudo systemctl enable docker
```

### `apt upgrade`時に docker のバージョンを変更させないようにする

※他のパッケージアップデート時につられて Docker のバージョンが上がり動作しなくなるのを防ぐためです。

```bash
sudo apt-mark hold docker.io
docker.io set on hold.
```

確認

```bash
apt-mark showhold
docker.io
```

→ OK！

### バージョン確認

```bash
docker --version
Docker version 17.12.1-ce, build 7390fc6
```

### Hello World

以下がコンソールに表示されれば Docker のセットアップは完了です。

```bash
docker run --rm hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

## docker-compose をインストール

### 最新のバージョンを確認

[こちら](https://github.com/docker/compose/blob/master/CHANGELOG.md) から最新のバージョンを確認する。

※2020/01/02 現在では`1.25.0`が最新のバージョンです。

### ダウンロード

`/usr/local/bin`配下にダウンロード

```bash
sudo curl -L https://github.com/docker/compose/releases/download/1.25.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

### 実行権限の付与

```bash
sudo chmod 0755 /usr/local/bin/docker-compose
```

### 確認

```bash
docker-compose --version
docker-compose version 1.25.0, build 0a186604
```

### 適当なサービスを起動してみる

```bash
mkdir -p docker/docker-services
cd docker/docker-services
vi docker-compose.yml
```

```yaml
version: "3"
volumes:
  db_data:
services:
  database:
    image: postgres:11.6
    container_name: postgres
    ports:
      - 5432:5432
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8"
```

起動時に以下のエラーが発生

```bash
docker-compose up
Creating network "postgres_default" with the default driver
ERROR: Failed to Setup IP tables: Unable to enable NAT rule:  (iptables failed: iptables --wait -t nat -I POSTROUTING -s 172.18.0.0/16 ! -o br-1fdb62035eeb -j MASQUERADE: iptables: Invalid argument. Run `dmesg' for more information.
 (exit status 1))
```

WSL のネットワーク周りの対応が完全でないことが原因とのこと。

WSL2 であれば解決するらしいので、WSL → WSL2 へと切り替えを行う。

## WSL2 への切り替え

と、いうことで[WSL2 で docker-compose を使えるようにするまで](https://qiita.com/suaaa7/items/744f58319c04d9b6bfbe)を参考に作業を進める。

手順は、

1. Windows 10 Insider Preview の登録
   - [https://insider.windows.com/ja-jp/getting-started/#register](https://insider.windows.com/ja-jp/getting-started/#register)から Windows 10 Insider Preview を登録（要 Microsoft アカウント）
2. Windows 10 Preview Build のインストール
   - 設定 > 更新とセキュリティ > Windows Insider Program
   - Windows Update を実行する（結構時間がかかります。）
     ※Build Version が 18917 以降の場合は必要なし
3. WSL 2 のインストール
4. WSL 1 で Ubuntu のインストール（実施済みのため省略）
5. WSL 1 -> WSL 2 への切り替え
6. docker のインストール（実施済みのため省略）
7. docker-compose のインストール（実施済みのため省略）

とのこと。（1, 2 は省略します。）

### 3. WSL2 のインストール

Power Shell を管理者権限で起動後、以下のコマンドを発行。（実行後再起動が必要）

```bash
PS C:\WINDOWS\system32> Enable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform
```

再起動後、上記コマンドを再実行し以下の状態となれば OK

```bash
PS C:\WINDOWS\system32> Enable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform

Path          :
Online        : True
RestartNeeded : False
```

### 5. WSL1 -> WSL2 へ切り替え

現在のバージョンを確認

```bash
PS C:\WINDOWS\system32> wsl -l -v
  NAME            STATE           VERSION
* Ubuntu-18.04    Stopped         1
```

バージョンの切り替え

```bash
PS C:\WINDOWS\system32> wsl --set-version Ubuntu-18.04 2
変換中です。この処理には数分かかることがあります...
WSL 2 との主な違いについては、https://aka.ms/wsl2 を参照してください
変換が完了しました。
```

6, 7 については前工程にて実施済みのため省略します。

### （今度こそ...）動作確認

```bash
docker-compose up
Creating network "postgres_default" with the default driver
Creating volume "postgres_db_data" with default driver
Pulling database (postgres:11.6)...
11.6: Pulling from library/postgres
804555ee0376: Pull complete
dc6ae8802d84: Pull complete
395ccdd34b05: Pull complete
a97aec38ba66: Pull complete
38ca37422b05: Pull complete
4f48902af5dd: Pull complete
f3aa2278d16c: Pull complete
babec8f73680: Pull complete
1b9e5f5edfb8: Pull complete
d72f7887945f: Pull complete
f16aa76dd9d7: Pull complete
71fe97949a0b: Pull complete
b8f647bf25fd: Pull complete
6598592ebb18: Pull complete
Digest: sha256:3695eeddceb287a15540881344b846c07c53518010d842e2ab0103f041a2412e
Status: Downloaded newer image for postgres:11.6
Creating postgres ... done
Attaching to postgres
postgres    | The files belonging to this database system will be owned by user "postgres".
postgres    | This user must also own the server process.
postgres    |
postgres    | The database cluster will be initialized with locale "en_US.utf8".
postgres    | The default text search configuration will be set to "english".
postgres    |
postgres    | Data page checksums are disabled.
postgres    |
postgres    | fixing permissions on existing directory /var/lib/postgresql/data ... ok
postgres    | creating subdirectories ... ok
postgres    | selecting default max_connections ... 100
postgres    | selecting default shared_buffers ... 128MB
postgres    | selecting default timezone ... Etc/UTC
postgres    | selecting dynamic shared memory implementation ... posix
postgres    | creating configuration files ... ok
postgres    | running bootstrap script ... ok
postgres    | performing post-bootstrap initialization ... ok
postgres    | syncing data to disk ... ok
postgres    |
postgres    | Success. You can now start the database server using:
postgres    |
postgres    |     pg_ctl -D /var/lib/postgresql/data -l logfile start
postgres    |
postgres    |
postgres    | WARNING: enabling "trust" authentication for local connections
postgres    | You can change this by editing pg_hba.conf or using the option -A, or
postgres    | --auth-local and --auth-host, the next time you run initdb.
postgres    | waiting for server to start....2020-01-04 14:01:29.863 UTC [48] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
postgres    | 2020-01-04 14:01:29.891 UTC [49] LOG:  database system was shut down at 2020-01-04 14:01:29 UTC
postgres    | 2020-01-04 14:01:29.899 UTC [48] LOG:  database system is ready to accept connections
postgres    |  done
postgres    | server started
postgres    |
postgres    | /usr/local/bin/docker-entrypoint.sh: ignoring /docker-entrypoint-initdb.d/*
postgres    |
postgres    | waiting for server to shut down...2020-01-04 14:01:29.947 UTC [48] LOG:  received fast shutdown request
postgres    | .2020-01-04 14:01:29.952 UTC [48] LOG:  aborting any active transactions
postgres    | 2020-01-04 14:01:29.954 UTC [48] LOG:  background worker "logical replication launcher" (PID 55) exited with exit code 1
postgres    | 2020-01-04 14:01:29.954 UTC [50] LOG:  shutting down
postgres    | 2020-01-04 14:01:29.989 UTC [48] LOG:  database system is shut down
postgres    |  done
postgres    | server stopped
postgres    |
postgres    | PostgreSQL init process complete; ready for start up.
postgres    |
postgres    | 2020-01-04 14:01:30.062 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
postgres    | 2020-01-04 14:01:30.062 UTC [1] LOG:  listening on IPv6 address "::", port 5432
postgres    | 2020-01-04 14:01:30.070 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
postgres    | 2020-01-04 14:01:30.092 UTC [57] LOG:  database system was shut down at 2020-01-04 14:01:29 UTC
postgres    | 2020-01-04 14:01:30.100 UTC [1] LOG:  database system is ready to accept connections
```

→ OK！

一応確認

```bash
docker exec -it postgres bash
```

```bash
root@a48d26e23174:/# psql -U postgres postgres
psql (11.6 (Debian 11.6-1.pgdg90+1))
Type "help" for help.

postgres=#
```

→ OK！

# 参考

- [Windows 10 で Linux プログラムを利用可能にする WSL をインストールする（バージョン 1803 以降対応版）](https://www.atmarkit.co.jp/ait/articles/1608/08/news039.html)

- [docker-compose リリースノート](https://github.com/docker/compose/blob/master/CHANGELOG.md)

- [WSL2 で docker-compose を使えるようにするまで](https://qiita.com/suaaa7/items/744f58319c04d9b6bfbe)
