title: "Snappy Ubuntu Core on IDCFクラウド - Part1: 公式OVAを使う"
date: 2015-02-04 01:11:39
tags:
 - IDCFクラウド
 - SnappyUbuntuCore
 - cloud-config
 - idcf-compute-api
description: 2015-01-14に待望のSnappy Ubuntu CoreのOVAイメージが公開されました。以前はVMwareにデプロイするときはかなり面倒で、次のステップが必要でした。1. qemu-imgコマンドでイメージをQCOW2形式からVMDK形式する 2. VMXファイルをマニュアルで記述する 3. ovftoolコマンドでVMXファイルからOVAを作成する 公式のOVAにはcloud-initもインストール済みなので、cloud-config.ymlをuserdataとして使えば、IDCFクラウドでもSnappy Ubuntu Coreが簡単に使えます。

---

2015-01-14に待望のSnappy Ubuntu Coreの[OVAイメージが公開](https://insights.ubuntu.com/2015/01/15/snappy-ubuntu-core-now-on-the-hypervisor-of-your-choice-with-ova/)されました。以前はVMwareにデプロイするときはかなり面倒で、野良OVAイメージのビルドに次のステップが必要でした。

1. qemu-imgコマンドでQCOW2イメージをVMDK形式する
2. VMXファイルをマニュアルで記述する
3. ovftoolコマンドでVMXファイルからOVAイメージを作成する

公式のOVAにはcloud-initもインストール済みなので、cloud-config.ymlをuserdataとして使えば、IDCFクラウドでもSnappy Ubuntu Coreが簡単に使えます。

<!-- more -->

## OVAテンプレートの作成

[Snappy Ubuntu Core now on the hypervisor of your choice with OVA](https://insights.ubuntu.com/2015/01/15/snappy-ubuntu-core-now-on-the-hypervisor-of-your-choice-with-ova/)のページにある[latest OVA image](http://cloud-images.ubuntu.com/snappy/devel/core/current/devel-core-amd64-cloud.ova)のリンクからOVAのURLを取得します。テンプレート名、説明、URL、OSタイプ以外はデフォルトです。OSタイプは一番近い`Ubuntu 12.04 (64bit)`を選びました。

* テンプレート名: 任意
* 説明: 任意
* URL: http://cloud-images.ubuntu.com/snappy/devel/core/current/devel-core-amd64-cloud.ova
* ゾーン: tesla
* ハイパーバイザー: VMware
* OSタイプ: Ubuntu 12.04 (64bit)
* フォーマット: OVA
* エクスポート: 有効
* パスワードリセット: 有効
* ダイナミックスケール: 有効
* ルートディスクコントローラ: scsi
* NICアダプタ: Vmxnet3
* キーボード: Japanese


## 仮想マシンの作成

### 最小設定のcloud-config

Snappy Ubuntu CoreはCoreOSと同じようにcloud-configを使って起動する必要があります。cloud-configを使わなくても起動はできますが、外部からのSSH接続が許可されていないため、コンソールからしか使えません。今回は最小設定のcloud-configを使います。

``` yaml cloud-config.yml
#cloud-config
snappy:
  ssh_enabled: True
```

### 必要な情報の取得

仮想マシンを作成する時に必要なパラメータをIDCFクラウドのCLIを使い確認します。

* listTemplates: テンプレートのidを取得
* listZones: ゾーンのidを取得
* listServiceOfferings: 仮想マシンサイズのidを取得
* listSSHKeyPairs: SSH公開鍵の名前を取得

今回利用するOVAのSnappy Ubuntu CoreにはCloud-initがインストールされています。IDCFクラウドが採用しているCloudStackをDataSourceとして自動的に認識してくれるので、指定した名前のSSH公開鍵をubuntuユーザーの`authorized_keys`に登録してくれます。これはかなり便利です。

またAPIを実行するために必要な以下の情報をポータルの画面から確認しておきます。

* `IDCF_COMPUTE_HOST`: エンドポイント
* `IDCF_COMPUTE_API_KEY`: APIキー
* `IDCF_COMPUTE_SECRET_KEY`: シークレットキー

### 仮想マシン作成のPythonスクリプト

IDCFクラウドのCLIツールはPythonで書いているので、Pythonプログラムのモジュールとしてimportできます。cloud-configはuserdataとしてBASE64でエンコードしてからパラメータに使います。


``` python deploy.py
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import base64
from idcf.compute import Compute

def main():
    rawuserdata = '''#cloud-config
snappy:
  ssh_enabled: True
'''
    userdata = base64.b64encode(rawuserdata)
    client = Compute(
        host='エンドポイント',
        api_key='APIキー',
        secret_key='シークレットキー',
        debug=True)

    retval = client.deployVirtualMachine(
        keypair='SSH公開鍵の名前',
        serviceofferingid='仮想マシンサイズのID',
        templateid='登録したOVAテンプレートのID',
        zoneid='ゾーンのID',
        name='任意のhostname',
        userdata=userdata)

    print(retval)

if __name__ == "__main__":
    main()
```

### Pythonスクリプトの実行

仮想マシンが作成されるまで数分待ちます。SSHサーバーが起動するまでに初回設定が入るため、ポータル画面で状態がRunningになっていてもSSHで接続できるまでさらに数分必要です。コンソールでlogin待機になるまでしばらく待ちます。

### SSH公開鍵認証でログイン

アカウント内の別の仮想マシンから、または、パブリックのIPアドレスをポートフォワードしてSSH公開鍵認証で接続します。

``` bash
$ ssh -A ubuntu@snappy 
Welcome to Ubuntu Vivid Vervet (development branch) (GNU/Linux 3.18.0-9-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

Welcome to the Ubuntu Core rolling development release.

 * See https://ubuntu.com/snappy

It's a brave new world here in snappy Ubuntu Core! This machine
does not use apt-get or deb packages. Please see 'snappy --help'
for app installation and transactional updates.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.
ubuntu@snappy:~$
```

### バージョン確認など

OSのバージョンを確認します。

``` bash
$ cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=15.04
DISTRIB_CODENAME=vivid
DISTRIB_DESCRIPTION="Ubuntu Vivid Vervet (development branch)"
```

cloud-initのバージョンは0.7.7です。

``` bash
$ cloud-init --version
cloud-init 0.7.7
```

ubuntu-coreはedge 144がインストールされていて最新の状態です。

``` bash
$ snappy versions
Part         Tag   Installed  Available  Fingerprint     Active
ubuntu-core  edge  144        -          395812b6615109  *
This command needs root, please run with sudo
$ snappy update-versions
0 updated components are available with your current stability settings.
```

snappyコマンドのバージョンです。

``` bash
$ snappy --version
0.1
```

まだframeworksもappsもインストールされていません。

``` bash
$ snappy info
release: ubuntu-core/devel
frameworks:
apps:
```
### Dockerをインストール

とりあえずDockerをインストールします。Snappy Ubuntu CoreではDockerのインストールはとても簡単なのですぐに使い始めることができます。

``` bash
$ snappy search docker
Part    Version    Description
docker  1.3.3.001  The docker app deployment mechanism
$ sudo snappy install docker
docker      4 MB     [===================================================================]    OK
Part    Tag   Installed  Available  Fingerprint     Active
docker  edge  1.3.3.001  -          60b98945e5fc1d  *
Reboot to use the new ubuntu-core.
$ sudo snappy versions
Part         Tag   Installed  Available  Fingerprint     Active
ubuntu-core  edge  144        -          395812b6615109  *
docker       edge  1.3.3.001  -          60b98945e5fc1d  *
Reboot to use the new ubuntu-core.
```

### アプリのインストール

サンプルのhello-worldアプリをインストールします。

``` bash
$ sudo snappy install hello-world
hello-world     31 kB     [==============================================================]    OK
Part         Tag   Installed  Available  Fingerprint     Active
hello-world  edge  1.0.5      -          3e8b192e18907d  *
Reboot to use the new ubuntu-core.
```

先ほどのdockerはframeworks、hello-worldアプリはappsとしてレイアで区別されます。

``` bash
$ snappy info
release: ubuntu-core/devel
frameworks: docker
apps: hello-world
```
Hello World!を表示するだけのアプリがインストールされました。
 
``` bash
$ which hello-world.echo
/home/ubuntu/snappy-bin/hello-world.echo
$ hello-world.echo
Hello World!
```