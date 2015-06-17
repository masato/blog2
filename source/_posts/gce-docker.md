title: 'GCEにDockerをインストール'
date: 2014-05-13 14:58:33
tags:
 - GCE
 - Debian
 - Docker
description: 前回はGCEにCoreOSのクラスタを構築しました。CoreOS以外のOSでもDockerを試してみます。開設されたばかりのアジアリージョンを使ってみます。
---
[前回](/2014/05/12/gce-coreos-cluster-prepare/)はGCEにCoreOSのクラスタを構築しました。
CoreOS以外のOSでもDockerを試してみます。開設されたばかりのアジアリージョンを使ってみます。

<!-- more -->

### Google Cloud SDK をインストール

インストーラーを実行します。

``` bash
$ curl https://dl.google.com/dl/cloudsdk/release/install_google_cloud_sdk.bash | bash
$ source ~/.bashrc
```

出力されるURLをブラウザで実行して、表示された認証コードを入力します。

``` bash
$ gcloud auth login
Enter verification code:
```

デフォルトのプロジェクトIDを設定します。

``` bash
$ gcloud config set project DEFAULT_PROJECT_ID
```

設定の確認をします。

``` bash
$ gcloud config list
```

### インスタンスの作成

最新のDebianバックポートのインスタンスを作成します。Andromedaを享受できるカーネルドライバが入っているそうです。
20秒くらいでインスタンスが作成されました。はやい！
``` bash
$ gcutil addinstance spike-docker --persistent_boot_disk --zone=asia-east1-a --machine_type=n1-standard-1 --image=projects/debian-cloud/global/images/backports-debian-7-wheezy-v20140415 
```

ローカルマシンの環境変数をSSHで送信しないよう、コメントアウトします。

``` bash
$ sudo vi /etc/ssh/ssh_config
# SendEnv LANG LC_*
```

gcutilコマンドで、Vagrantみたいに簡単にSSH接続ができます。

``` bash
$ gcutil ssh spike-docker
gce@spike-docker:~$
```

### Dockerのインストール

インストーラーを実行します。

``` bash
$ curl get.docker.io | bash
```

作業ユーザーをdockerグループに追加して、sudoなしでdockerコマンドが実行できるようにします。

``` bash
$ sudo usermod -aG docker gce
$ exit
```

### コンテナの実行

SSH接続します。

``` bash
$ gcutil ssh spike-docker
```

自動起動の設定をします。

``` bash
$ sudo insserv -d docker
```

コンテナを実行します。

``` bash
$ docker run busybox echo 'hello docker'
hello docker
```

### まとめ

GCEはインスタンスの起動がはやいので、新しいセットのサーバーの構成を試すときに適しています。
gcutilも使い勝手がよいのですが、引数が長くなりがちなのでsalt-cloudを使うとよいかも。
次は、CoreOSとは違う方法でDockerクラスタをSerfで構築してみます。

