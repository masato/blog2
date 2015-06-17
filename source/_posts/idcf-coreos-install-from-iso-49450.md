title: 'IDCFクラウドにStableのCoreOS 494.5.0をISOからインストールする'
date: 2014-12-19 23:03:11
tags:
 - CoreOS
 - Couchbase
 - IDCFクラウド
 - idcf-compute-api
 - jq
description: 久しぶりにIDCFクラウドにCoreOSをISOからインストールします。手順は以前と同じです。今回の用途はCouchbase Serverクラスタを構築するため、AWS CloudFormationのcloud-configを参考にしてインストールします。最近はetcdを専用に1つしか作らないことが多かったですが、通常通りにすべてのCoreOSの仮想マシンにetcdをインストールします。
---

久しぶりにIDCFクラウドにCoreOSをISOからインストールします。手順は[以前](/2014/08/18/idcf-coreos-install-from-iso/)と同じです。今回の用途はCouchbase Serverクラスタを構築するため、AWS CloudFormationの[cloud-config](http://tleyden-misc.s3.amazonaws.com/couchbase-coreos/coreos-stable-pv.template)を参考にしてインストールします。最近はetcdを専用に1つしか作らないことが多かったですが、通常通りにすべてのCoreOSの仮想マシンにetcdをインストールします。

<!-- more -->

### Couchbase Server用のCoreOSクラスタ

CoreOSの使い方はCouchbase LabsオフィシャルのDockerイメージを参考にします。

* [Running Couchbase Cluster Under CoreOS on AWS](http://tleyden.github.io/blog/2014/11/01/running-couchbase-cluster-under-coreos-on-aws/)
* [tleyden5iwx/couchbase-server-3.0.1](https://registry.hub.docker.com/u/tleyden5iwx/couchbase-server-3.0.1/)
* [couchbase-server-docker](https://github.com/couchbaselabs/couchbase-server-docker)


### ISOの登録について

現在の[CoreOSのISO](https://coreos.com/docs/running-coreos/platforms/iso/)は、Stableの494.5.0でした。まずIDCFクラウドにISOを登録してからCoreOSの仮想マシンを作成します。

* ISO名: coreos_49450
* 説明: coreos_49450
* URL: http://stable.release.core-os.net/amd64-usr/current/coreos_production_iso_image.iso
* ゾーン: tesla
* OSタイプ: Other (64-bit)
* エクスポート: 有効
* ブータブル: 有効

### idcf-compute-apiを使ってISOをアップロードする場合

すべてIDCFクラウドのポータル画面から操作できますが、なるべくCLIを使いたいです。ISOをアップロードするところをCLIを使って実行します。

ゾーンの確認をします。

``` bash
$ idcf-compute-api listZones -t=id,name
+--------------------------------------+-------+
| id                                   | name  |
+--------------------------------------+-------+
| a117e75f-d02e-4074-806d-889c61261394 | tesla |
+--------------------------------------+-------+
```

OSタイプを確認します。

``` bash
$ idcf-compute-api listOsTypes | jq '.listostypesresponse | .ostype[] | select(contains({description:"Other (64-bit)"}))'
{
  "description": "Other (64-bit)",
  "id": "c213145e-4ada-11e4-bd06-005056812ba5",
  "oscategoryid": "c1b5fc37-4ada-11e4-bd06-005056812ba5"
}
```

CoreOS 494.5.0のISOをアップロードします。

``` bash
$ idcf-compute-api registerIso \
  --name=coreos_49450  \
  --displaytext=coreos_49450 \
  --url=http://stable.release.core-os.net/amd64-usr/current/coreos_production_iso_image.iso \
  --zoneid=a117e75f-d02e-4074-806d-889c61261394 \
  --ostypeid=c213145e-4ada-11e4-bd06-005056812ba5 \
  --bootable=true
```

アップロードしたISOを確認します。

``` bash
$ idcf-compute-api listIsos -t=id,name,bootable,isready
+--------------------------------------+--------------+----------+---------+
| id                                   | name         | bootable | isready |
+--------------------------------------+--------------+----------+---------+
| 327e3145-0679-4c22-8a06-f1abb7e224cf | coreos_49450 | True     | True    |
+--------------------------------------+--------------+----------+---------+
```

### CoreOSの仮想マシンを作成する

IDCFクラウドのポータル画面から操作して、アップロードしたISOからCoreOSの仮想マシンを作成します。ISOから起動した仮装マシンにはパスワード認証をするためSSH Keyは指定しません。Running状態になったら、コンソ-ルを開きます。ブラウザで開いたコンソールから、coreユーザーにパスワードを付けてSSH接続でパスワード認証ができるようにします。

``` bash
$ sudo passwd core
```

### CoreOSをディスクにインストールする

さきほどコンソールからパスワードを設定したのでパスワードでSSH接続が可能になっています。ISOから起動した仮想マシンはLive DVDから起動したLinuxと同じでディスクに永続化されません。ディスクにCoreOSをインストールしてから使います。

``` bash
$ ssh core@10.3.0.47 -o PreferredAuthentications=password
```

[AWS CloudFormationのテンプレート](http://tleyden-misc.s3.amazonaws.com/couchbase-coreos/coreos-stable-pv.template)を参考にして、cloud-config.ymlを作成します。

discoveryは、https://discovery.etcd.io/newを実行して、新規に作成します。addrとpeer-addrはCoreOSクラスタは同一のVLAN内に構築するため、仮想マシンのプライベートIPアドレスを指定します。ssh_authorized_keysは任意の公開鍵を指定します。

`write_files`の`/etc/environment`以外の3つがCouchbase Server用のファイルです。Couchbase Serverを使わない場合はこの3つは不要です。

``` yaml ~/cloud-config.yml
#cloud-config
write_files:
  - path: /etc/environment
    content: |
      COREOS_PUBLIC_IPV4=
      COREOS_PRIVATE_IPV4=10.3.0.47
  - path: /etc/systemd/system/docker.service.d/increase-ulimit.conf
    owner: core:core
    permissions: 0644
    content: |
      [Service]
      LimitMEMLOCK=infinity
  - path: /var/lib/couchbase/data/.README
    owner: core:core
    permissions: 0644
    content: |
      Couchbase Data files are stored here
  - path: /var/lib/couchbase/index/.README
    owner: core:core
    permissions: 0644
    content: |
      Couchbase Index files are stored here
coreos:
  etcd:
    discovery: https://discovery.etcd.io/d7bd532d2f3b86b050515bf3bdc1d93d
    addr: 10.3.0.47:4001
    peer-addr: 10.3.0.47:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
    - name: docker.service
      command: restart
    - name: timezone.service
      command: start
      content: |
        [Unit]
        Description=timezone
        [Service]
        Type=oneshot
        RemainAfterExit=yes
        ExecStart=/usr/bin/ln -sf ../usr/share/zoneinfo/Japan /etc/localtime
write_files:
ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC...
```

cloud-conig.ymlを使って、ディスクにCoreOSをインストールし直します。

``` bash
$ sudo coreos-install -d /dev/sda -C stable -c ./cloud-config.yml
$ sodo reboot
```

reboot後、公開鍵を使ってSSH接続ができるようになります。

``` bash
$ ssh -A core@10.3.0.47
```

### 確認

cloud-config.ymlで`write_files`したファイルを確認します。

``` bash
$ cat /var/lib/couchbase/data/.README
Couchbase Data files are stored here
```

fleetctlでfleetクラスタに仮装マシンが追加されたことを確認します。

``` bash
$ fleetctl list-machines
MACHINE         IP              METADATA
f0f5fcc4...     10.3.0.47       -
```

最後にIDCFクラウドのポータルから仮想マシンを停止して、ISOをデタッチします。
