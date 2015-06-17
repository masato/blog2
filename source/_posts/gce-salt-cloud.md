title: 'SaltでGCEにプロビジョニング'
date: 2014-05-15 03:47:31
tags:
 - GCE
 - Salt
 - salt-cloud
description: GCEのパートナーには、CoreOSやQubole,Saltなど興味深いサービスがたくさんあります。同じPythonで書かれているAnsibleもよいですが、最近気に入っているSaltを使ってGCEにプロビジョニングをしてみます。
---

GCEの[パートナー](https://cloud.google.com/partners/)には、[CoreOS](https://coreos.com/)や[Qubole](http://www.qubole.com/),[Salt](http://www.saltstack.com/)など興味深いサービスがたくさんあります。同じ
Pythonで書かれている[Ansible](http://www.ansible.com/home)もよいですが、最近気に入っているSaltを使ってGCEにプロビジョニングをしてみます。

<!-- more -->

### GCE用のユーザー作成

GCEを操作するユーザーを作成します。gcutil sshコマンドの挙動はちょっと変わっていて、慣れるまで専用のユーザーで試してみます。

gceuserを作成します。

``` bash
$ sudo adduser gceuser
$ sudo su - gceuser
```

Google Cloud SDKのインストールをします。

``` bash
$ curl https://dl.google.com/dl/cloudsdk/release/install_google_cloud_sdk.bash | bash
$ source ~/.bashrc
$ gcloud auth login
$ gcloud config set project DEFAULT_PROJECT_ID
$ gcloud config list
```

### Salt Masterの作成

Salt Master用インスタンスを作成します。

``` bash
$ gcutil addinstance salt --persistent_boot_disk --zone=asia-east1-a --machine_type=n1-standard-1 --image=debian-7
```

SSH接続します。

``` bash
$ gcutil ssh salt
```

/etc/hostsから、Salt Master用インスタンスのFQDNを確認します。

``` bash
$ cat /etc/hosts
127.0.0.1       localhost
::1             localhost ip6-localhost ip6-loopback
fe00::0         ip6-localnet
ff00::0         ip6-mcastprefix
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

10.240.157.xxx salt.c.xxx-xxx-xxx.internal salt  # Added by Google
```

Salt Masterをインストールします。

``` bash
$ curl -L http://bootstrap.saltstack.org | sudo sh -s -- -M -N git develop
```

salt-cloudで使う、libcloudをインストールします。

``` bash
$ sudo pip install apache-libcloud
```

### GCE用クライアントIDの作成

[Google Developers Console](https://console.developers.google.com/project)へログインし、現在使っているプロジェクトIDを開きます。
APIと認証 -> 認証情報　-> 新しいクライアントIDを作成 -> サービスアカウント -> と進み、クライアントIDを作成します。
salt-cloudで使うサービスアカウントの秘密鍵をダウンロードします。xxx-privatekey.p12の名前で保存します。


サービスアカウントのメールアドレスは、後でservice_account_email_addressとしてgce.confに指定します。
サービスアカウントの秘密鍵とgcuserのgoogle_compute_engineの鍵を、salt-master用インスタンスにコピーします。

``` bash
$ sudo cp ~/xxx-privatekey.p12 /home/gceuser
$ gcutil push salt ~/xxx-privatekey.p12 /home/gceuser
$ gcutil push salt ~/.ssh/google_compute_engine /home/gceuser
```

サービスアカウントの鍵を[PKCS12フォーマット](http://en.wikipedia.org/wiki/PKCS_12)から、libcloud用に作り直します。

``` bash
$ gcutil ssh salt
$ sudo mkdir /etc/cloud
$ sudo cp ~/google_compute_engine /etc/cloud/
$ openssl pkcs12 -in  ~/xxx-privatekey.p12 -passin pass:notasecret \
-nodes -nocerts |sudo openssl rsa -out  /etc/cloud/service_account_private_key
MAC verified OK
writing RSA key
```

### Salt Masterで GCE用の設定

クラウドプロバイダーとクラウドプロファイルを保存するディレクトリを作成します。

``` bash
$ sudo mkdir /etc/salt/cloud.providers.d
$ sudo mkdir /etc/salt/cloud.profiles.d
```

GCEのプロバイダーファイルを作成します。`Salt Minion`用インスタンスでもgceuserを使うためssh_usernameを指定します。

``` yaml /etc/salt/cloud.providers.d/gce.conf
my-gce-config:
  project: "DEFAULT_PROJECT_ID"
  service_account_email_address: "xxx@developer.gserviceaccount.com"
  service_account_private_key: "/etc/cloud/service_account_private_key"

  minion:
    master: salt.c.xxx-xxx-xxx.internal

  grains:
    node_type: broker
    release: 1.0.1
  ssh_username: gceuser
  ssh_keyfile: /etc/cloud/google_compute_engine
  provider: gce
```

GCEに作成するインスタンスのプロファイルを指定します。

``` yaml /etc/salt/cloud.profiles.d/gce.conf
debian-7:
  image: debian-7
  size: n1-standard-1
  location: asia-east1-a
  network: default
  tags: '["one", "two", "three"]'
  metadata: '{"one": "1", "2": "two"}'
  use_persistent_disk: True
  delete_boot_pd: False
  deploy: True
  make_master: False
  provider: my-gce-config
```

### Salt Minion用インスタンスの作成

salt-cloudコマンドを使い、GCE上にインスタンスを作成します。コマンド実行にはsudoが必用です。

``` bash
$ sudo salt-cloud -p debian-7 minion1
[INFO    ] salt-cloud starting
[INFO    ] Creating GCE instance minion1 in asia-east1-a
[INFO    ] Rendering deploy script: /usr/lib/python2.7/dist-packages/salt/cloud/deploy/bootstrap-salt.sh
Warning: Permanently added '107.167.182.xxx' (ECDSA) to the list of known hosts.
 *  INFO: /bin/sh /tmp/.saltcloud/deploy.sh -- Version 2014.04.16
 *  INFO: System Information:
 *  INFO:   CPU:          GenuineIntel
 *  INFO:   CPU Arch:     x86_64
 *  INFO:   OS Name:      Linux
 *  INFO:   OS Version:   3.2.0-4-amd64
 *  INFO:   Distribution: Debian 7.4

 *  INFO: Installing minion
...
```

### saltコマンドのテスト

saltコマンドで、作成した`Salt Minion`インスタンスへ、pingを打ちます。

``` bash
$ sudo salt 'minion*' test.ping
minion1:
    True
```

### まとめ

GCE上にSaltからプロビジョニングと構成管理ができる環境構築までできました。
もう一つのSaktの機能であるリモート実行を使い、長くなりがちなdockerコマンドを、saltコマンドでまとめてみようと思います。
`Salt Minion`のインスタンスの作成とDockerのインストール、コンテナの起動までSaltを使ってみます。