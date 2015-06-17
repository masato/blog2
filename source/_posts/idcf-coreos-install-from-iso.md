title: 'IDCFクラウドでCoreOSをISOからインストールする'
date: 2014-08-18 19:05:00
tags:
 - CoreOS
 - IDCFクラウド
 - Panamax
 - idcf-compute-api
 - jq
description: IDCFクラウドにCoreOSクラスタを構築する - Part5 再セットアップなどではボリュームを作業マシンにアタッチしてインストール作業をしました。CoreOSはBootableのISOも提供されているので、今回はBooting CoreOS from an ISOを読みながらIDCFクラウドで利用してみます。IDCFクラウドにCoreOSクラスタを構築する - Part4 367.1.0のissueで確認しましたが、CoreOSのStable Channelのcurrentバージョンは367.1.0です。今回はPanamaxをインストールするためにCoreOSを用意します。Installing Panamaxを読むと、推奨バージョンは367.1.0なので都合が良いです。
---

* `Update 2014-08-19`: [Panamax in CoreOS on IDCFクラウド - Part1: インストール](/2014/08/19/panamax-in-coreos-on-idcf/)

[IDCFクラウドにCoreOSクラスタを構築する - Part5: 再セットアップ](/2014/07/19/idcf-coreos-cluster-re-setting-up/)などではボリュームを作業マシンにアタッチしてインストール作業をしました。
CoreOSはBootableのISOも提供されているので、今回は[Booting CoreOS from an ISO](https://coreos.com/docs/running-coreos/platforms/iso/)を読みながらIDCFクラウドで利用してみます。

[IDCFクラウドにCoreOSクラスタを構築する - Part4: 367.1.0のissue](/2014/07/17/idcf-coreos-cluster-36710/)で確認しましたが、CoreOSの`Stable Channel`のcurrentバージョンは`367.1.0`です。

今回は[Panamax](http://panamax.io/)をインストールするためにCoreOSを用意します。
[Installing Panamax](https://github.com/CenturyLinkLabs/panamax-ui/wiki/Installing-Panamax)を読むと、推奨バージョンは`367.1.0`なので都合が良いです。


<!-- more -->


### idcf-compute-apiのインストール

Pythonで書かれたIDCFクラウドCLIをインストールします。
作業マシンのPythonバージョン確認すると`2.7.6`なのでPythonzを使わず、普通にvirtualenvします。

```
$ python -V
Python 2.7.6
```

virtualenvをインストールしてactivateします。

```
$ sudo apt-get update
$ sudo apt-get install python-setuptools
$ sudo easy_install pip
$ sudo pip install virtualenv
$ virtualenv ~/.venv/Python-2.7.6 && source ~/.venv/Python-2.7.6/bin/activate
```

idcf-compute-apiをインストールします。

```
$ sudo apt-get install python-dev zlib1g-dev libxml2-dev libxslt-dev
$ pip install idcf.compute-0.11.1.tar.gz
$ vi ~/.idcfrc
$ chmod 600 ~/.idcfrc
$ idcf-compute-api listZones -t=id,name
```

環境設定ファイルを書きます。`API_KEY`などの設定情報は、IDCFクラウドのポータルにログインして確認します。

```
$ vi ~/.idcfrc
[account]
host={END_POINT} 
api_key={APIKEY} 
secret_key={SECRET_KEY} 
$ chmod 600 ~/.idcfrc
```

インストールの確認をします。

```
$ idcf-compute-api listZones -c=id,name
"id","name"
1,"jp-east-t1v"
2,"jp-east-f2v"
```

### jqとwaitjobコマンドの用意

jqをインストールします。

```
$ wget http://stedolan.github.io/jq/download/linux32/jq -P ~/bin
$ chmod u+x ~/bin/jq
```

idcf-compute-apiの非同期コマンドのステータスをポーリングするコマンドを用意します。

``` bash ~/bin/waitjob
#!/usr/bin/env bash
set -eo pipefail

while :
do
  json=$(idcf-compute-api queryAsyncJobResult --jobid=$1)
  status=$(echo ${json} | jq '.queryasyncjobresultresponse.jobstatus')
  if [ ${status} -eq 0 ]; then
    echo -ne "."
    sleep 10s
  else
    echo -e "\n"
    echo ${json} | jq ".queryasyncjobresultresponse | {jobid: $1,jobresult}"
    break
  fi
done
```

実行権限をつけます。~/binをパスに通すため一度exitします。

``` bash
$ chmod u+x ~/bin/jq
$ chmod u+x ~/bin/waitjob
$ exit
```

### ISOの登録

`Stable Channel`からISOをダウンロードURLをコピーして、IDCFクラウドにISOを登録します。
[registerIso](http://www.idcf.jp/cloud/docs/api/user/registerIso)コマンドを利用します。133MBなのでISOのダウンロードは速くに終了します。

```
$ idcf-compute-api registerIso \
  --name=coreos_36710  \
  --displaytext=coreos_36710 \
  --url=http://stable.release.core-os.net/amd64-usr/current/coreos_production_iso_image.iso \
  --zoneid=1 \
  --ostypeid=99 \
  --bootable=true
```

### ISOからインスタンスの作成

ISOからインスタンスはポータルの画面から作成します。
インスタンスが起動した後、コンソールを開くとcoreユーザーでログインした状態になっています。

SSHでパスワード接続をするため、coreユーザーにパスワードを設定します。

```
$ sudo passwd core
```

### CoreOSのインスタンスにSSH接続

作業マシンから、パスワードを使いSSH接続をします。

```
$ ssh core@10.1.0.66
CoreOS (beta)
Update Strategy: No Reboots
```

### CoreOS用の鍵作成

作業マシンでSSH接続に使うキーペアを作成します。あとでParamaxのインストールにも使いたいので、パスフレーズなしにします。

```
$ ssh-keygen -q -t rsa -f ~/.ssh/deis -N '' -C deis
```

### CoreOSのインストール

ipコマンドでネットワークを確認します。[Note](https://coreos.com/docs/running-coreos/bare-metal/installing-to-disk/)にあるように、cloud-config.ymlの`$private_ipv4`の箇所は実際のIPアドレスに変更します。

```
$ ip a
...
2: ens32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 02:00:3a:28:01:11 brd ff:ff:ff:ff:ff:ff
    inet 10.1.0.66/22 brd 10.1.3.255 scope global ens32
       valid_lft forever preferred_lft forever
    inet6 fe80::3aff:fe28:111/64 scope link
       valid_lft forever preferred_lft forever
```

cloud-configを書いていきます。
`/etc/environment`の`COREOS_PRIVATE_IPV4`に、上記で確認したDHCPで割り当てられたIPアドレスを記入します。
nsenterのラッパーも用意しておきます。

```  cloud-config.yml
#cloud-config

ssh_authorized_keys:
  - ssh-rsa AAAAB3Nz... deis

coreos:
  etcd:
    discovery: https://discovery.etcd.io/3e93233e53a5296af44307a6e9b3718e
    addr: 10.1.0.66:4001
    peer-addr: 10.1.0.66:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
    - name: stop-update-engine.service
      command: start
      content: |
        [Unit]
        Description=stop update-engine

        [Service]
        Type=oneshot
        ExecStart=/usr/bin/systemctl stop update-engine.service
        ExecStartPost=/usr/bin/systemctl mask update-engine.service

write_files:
  - path: /etc/environment
    permissions: 0644
    content: |
      COREOS_PUBLIC_IPV4=
      COREOS_PRIVATE_IPV4=10.1.0.66
  - path: /etc/profile.d/nse-function.sh
    permissions: 0755
    content: |
      function nse() {
        sudo nsenter --pid --uts --mount --ipc --net --target $(docker inspect --format="&#123;&#123; .State.Pid &#125;&#125;" $1)
      }

```

`Stable Channel`からCoreOSの`367.1.0`をインストールします。
BootableのISOから起動しているため、ディスクの/dev/sdaにインストールできます。

```
$ sudo coreos-install -d /dev/sda -C stable -c ./cloud-config.yml
...
Success! CoreOS stable 367.1.0 is installed on /dev/sda
```

rebootします。

```
$ sudo reboot
```

### 確認

作業マシンからSSH接続します。

```
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh -A core@10.1.0.66
CoreOS (stable)
```

### まとめ

ISOからインストールすると、ディスクにより簡単に短時間でインスタンスを作成できました。
これからはこの方法でCoreOSのインスタンスを用意していきます。

367.1.0のインスタンスが用意できたので、次はPanamaxのインストールをしてみます。
うまく動作したら、このバージョンでCoreOSクラスタはしばらく使っていこうと思います。


