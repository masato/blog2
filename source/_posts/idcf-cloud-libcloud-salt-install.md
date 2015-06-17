title: "libcloudのCloudStackドライバーからIDCFクラウドを使う"
date: 2014-11-24 21:11:07
tags:
 - IDCFクラウド
 - CloudStack
 - Salt
 - libcloud
 - idcf-compute-api
description: 以前、IDCFクラウドのCLIでSaltをプロビジョニングするの記事を書いたときIDCFクラウドのCloudStackのバージョンは2.2.15でした。2014年10月にリリースされた新しいIDCFクラウドは、CloudStackのバージョンが4.3に上がっています。CloudStack 4.xになると、libcloudのCloudStackドライバーでサポートされるので使ってみようと思います。libcloudのdeploy_nodeメソッドを使うと、インスタンスの作成後に任意のスクリプトを実行することができます。IDCFクラウドではcloud-initがデフォルトでは使えないため、bootstrapを実行するための方法として使えます。
---

以前、[IDCFクラウドのCLIでSaltをプロビジョニングする](/2014/05/29/idcf-api-salt/)の記事を書いたときIDCFクラウドのCloudStackのバージョンは2.2.15でした。

``` xml
$ idcf-compute-api listZones -x |  grep cloud-stack-version
<listzonesresponse cloud-stack-version="2.2.15.20121121113057">
```
2014年10月にリリースされた新しいIDCFクラウドは、CloudStackのバージョンが4.3に上がっています。

``` xml
$ idcf-compute-api listZones -x |  grep cloud-stack-version
<listzonesresponse cloud-stack-version="4.3.0.1">
```

CloudStack 4.xになると、[libcloudのCloudStackドライバー](http://libcloud.readthedocs.org/en/latest/compute/drivers/cloudstack.html)でサポートされるので使ってみようと思います。

[libcloud](https://libcloud.apache.org/)の[deploy_node](https://libcloud.readthedocs.org/en/latest/compute/drivers/cloudstack.html#libcloud.compute.drivers.cloudstack.CloudStackNodeDriver.deploy_node)メソッドを使うと、インスタンスの作成後に任意のスクリプトを実行することができます。IDCFクラウドではcloud-initがデフォルトでは使えないため、bootstrapを実行するための代替手段になります。

<!-- more -->

### 接続情報の確認

IDCFクラウドの[コンソール](https://console.jp-east.idcfcloud.com/)にログインして、左メニューのAPIのリンクを開きます。次の3つの値をメモしておきます。

* エンドポイント
* API Key
* Secret Key

### 作業用インスタンス

今回のlibcloudの使い方では、インスタンスの作成後にプライベートIPアドレスでSSH接続します。そのためにアカウント内作業用のインスタンスを用意します。以下の手順ではUbuntu14.04を作業用インスタンスとしています。

``` bash
$ cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04.1 LTS"
```

### libcloudのインストール

作業用のUbuntuインスタンスにapt-getでPythonのビルド環境をインスト-ルしてから、libcloudとSSH接続に使うParamikoのモジュールインストールします。

``` bash
$ sudo apt-get update && apt-get -y install python-dev python-pip
$ sudo pip install paramiko apache-libcloud
```

Pythonやモジュールのバージョンは以下の通りです。

``` bash
$ python -V
Python 2.7.6
$ pip --version
pip 1.5.4 frm /usr/lib/python2.7/dist-packages (python 2.7)
$ pip search apache-libcloud
apache-libcloud           - A standard Python library that abstracts away differences among
                            multiple cloud provider APIs. For more information and documentation,
                            please see http://libcloud.apache.org
  INSTALLED: 0.16.0 (latest)
$ pip search paramiko
paramiko                  - SSH2 protocol library
  INSTALLED: 1.15.1 (latest)
```

コンソール画面でメモした値を環境変数にセットします。

``` bash
$ export IDCF_COMPUTE_API_KEY={API Key}
$ export IDCF_COMPUTE_SECRET_KEY={Secret Key}
$ export IDCF_COMPUTE_HOST=compute.jp-east.idcfcloud.com
$ export IDCF_SSH_KEY=my_key
```

### deploy.py

作業用のインスタンスに以下のような簡単なスクリプトを用意します。rootユーザーで実行し、IDCFクラウドに登録してある秘密鍵の名前が、`/root/.ssh/{キー名}`にあることを前提にしています。

``` bash
$ ls -al /root/.ssh/my_key
-rw------- 1 root root 1680 11月 24 23:00 /root/.ssh/my_key
```

master_bootstrapは、salt-masterのインストールスクリプト、minion_bootstrapは、salt-minionのインストールスクリプトを定義しています。

``` python deploy.py
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from libcloud.compute.types import Provider
from libcloud.compute.providers import get_driver
from libcloud.compute.deployment import ScriptDeployment
from libcloud.compute.deployment import MultiStepDeployment
from libcloud.compute.base import NodeAuthSSHKey

import os
import time

ACCESS_KEY=os.environ.get('IDCF_COMPUTE_API_KEY')
SECRET_KEY=os.environ.get('IDCF_COMPUTE_SECRET_KEY')
HOST=os.environ.get('IDCF_COMPUTE_HOST')
PATH='/client/api'
SSH_KEY='/root/.ssh/{0}'.format(os.environ.get('IDCF_SSH_KEY_NAME'))
SSH_IF='private_ips'

class Salt(object):

    def __init__(self):
        self.cls = get_driver(Provider.CLOUDSTACK)
        self.driver = self.cls(key=ACCESS_KEY,
                               secret=SECRET_KEY,
                               host=HOST,
                               path=PATH)
        self.default_offering()

    def default_offering(self):
        self.image = [i for i in self.driver.list_images()
                         if i.name == 'Ubuntu Server 14.04 LTS 64-bit'][0]
        self.size = [s for s in self.driver.list_sizes()
                         if s.name == 'light.S1'][0]
        self.keyname = [k for k in self.driver.list_key_pairs()
                         if k.name == os.environ.get('IDCF_SSH_KEY_NAME')][0].name

    def deploy(self,name,bootstrap):
        start = time.time()
        print('start {0}'.format(name))
        script = ScriptDeployment(bootstrap)
        msd = MultiStepDeployment([script])
        node = self.driver.deploy_node(name=name,
                                  image=self.image,
                                  size=self.size,
                                  ssh_key=SSH_KEY,
                                  ex_keyname=self.keyname,
                                  ssh_interface=SSH_IF,
                                  deploy=msd)
        end = time.time()
        elapsed = end - start
        print('end {0}, eplapsed: {1}'.format(name,elapsed))

def main():

    master_bootstrap = '''#!/bin/bash
curl -L http://bootstrap.saltstack.com | sh -s -- -M
'''
    minion_bootstrap = '''#!/bin/bash
curl -L http://bootstrap.saltstack.com | sh
'''
    salt = Salt()
    salt.deploy('salt',master_bootstrap)
    for i in range(1, 3):
        name = 'minion{0}'.format(i)
        salt.deploy(name,minion_bootstrap)

if __name__ == "__main__":
    
    main()
```

### スクリプトの実行 

master x 1, minion x 2の3台構成で、Saltクラスタを構築します。だいたい10分くらいかかります。

``` bash
$ ./deploy.py
start salt
end salt, eplapsed: 195.698220968
start minion1
end minion1, eplapsed: 180.300945044
start minion2
end minion2, eplapsed: 165.452461958
```

### salt-minionの承認

ここでは簡単にsalt-masterにSSH接続をして、minionの登録をしてみます。

``` bash
$ ssh -i .ssh/my_key salt
$ salt-key -L
Accepted Keys:
Unaccepted Keys:
minion1.cs256idcfcloud.internal
minion2.cs256idcfcloud.internal
salt.cs256idcfcloud.internal
Rejected Keys:
$ salt-key -A
The following keys are going to be accepted:
Unaccepted Keys:
minion1.cs256idcfcloud.internal
minion2.cs256idcfcloud.internal
salt.cs256idcfcloud.internal
Proceed? [n/Y] Y
Key for minion minion1.cs256idcfcloud.internal accepted.
Key for minion minion2.cs256idcfcloud.internal accepted.
Key for minion salt.cs256idcfcloud.internal accepted.
$ salt-key -L
Accepted Keys:
minion1.cs256idcfcloud.internal
minion2.cs256idcfcloud.internal
salt.cs256idcfcloud.internal
Unaccepted Keys:
Rejected Keys:
```


