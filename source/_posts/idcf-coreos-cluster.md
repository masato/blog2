title: 'IDCFクラウドにCoreOSクラスタを構築する - Part1: CoreOSの準備'
date: 2014-06-28 12:31:51
tags:
 - IDCFクラウド
 - CoreOS
 - cloud-config
 - idcf-compute-api
 - jq
description: IDCFクラウドへCoreOSをディスクへインストールできるようになったので、GCEにCoreOSクラスタで実験した方法で、IDCFクラウド上にCoreOSクラスタを構築してみます。CoreOSはインストールするときにcloud-config.ymlを参照して、SSH鍵の設定やsystemdの.serviceファイルを設定します。IDCFクラウドの場合は、作業マシンと外付けのディスクを用意してインストール済みのディスクを作成します。今回はInstalling CoreOS to Diskを読みながら、IDCFクラウドへのCoreOSディスクインストールを復習します。
---

* `Update 2014-06-28`: [IDCFクラウドにCoreOSクラスタを構築する - Part1: CoreOSの準備](/2014/06/28/idcf-coreos-cluster)
* `Update 2014-07-10`: [IDCFクラウドにCoreOSクラスタを構築する - Part2: CoreOSのディスカバリ](/2014/07/10/idcf-coreos-cluster-discovery)
* `Update 2014-07-15`: [IDCFクラウドにCoreOSクラスタを構築する - Part3: クラスタをセットアップ](/2014/07/15/idcf-coreos-cluster-setting-up)
* `Update 2014-07-17`: [IDCFクラウドにCoreOSクラスタを構築する - Part4: 367.1.0のissue](/2014/07/17/idcf-coreos-cluster-36710/)
* `Update 2014-07-19`: [IDCFクラウドにCoreOSクラスタを構築する - Part5: 再セットアップ](/2014/07/19/idcf-coreos-cluster-re-setting-up/)

IDCFクラウドへCoreOSを[ディスクへインストール](/2014/06/03/idcf-coreos-install-disk/)できるようになったので、
GCEに[CoreOSクラスタ](/2014/05/12/gce-coreos-cluster-prepare/)で実験した方法で、IDCFクラウド上にCoreOSクラスタを構築してみます。

CoreOSはインストールするときにcloud-config.ymlを参照して、SSH鍵の設定やsystemdの.serviceファイルを設定します。
IDCFクラウドの場合は、作業マシンと外付けのディスクを用意してインストール済みのディスクを作成します。


今回は[Installing CoreOS to Disk](https://coreos.com/docs/running-coreos/bare-metal/installing-to-disk/)を読みながら、IDCFクラウドへのCoreOS[ディスクインストール](/2014/06/03/idcf-coreos-install-disk/)を復習します。


### TL;DR

[IDCFクラウドにCoreOSクラスタを構築する - Part4: 367.1.0のissue](/2014/07/17/idcf-coreos-cluster-36710/)で、`/var/lib/coreos-install/user_data`を編集後にCoreOSをrebootすると、clud-config.ymlの再読込できました。

<!-- more -->


### 1台構成の場合

[Installing CoreOS to Disk](https://coreos.com/docs/running-coreos/bare-metal/installing-to-disk/)に従って、clud-config.ymlにインスタンスを構成する情報を記述していきます。

[前回](https://coreos.com/docs/running-coreos/bare-metal/installing-to-disk/)のcloud-config.ymlでは、必須のSSH鍵認証の設定だけ行っていました。

``` yml clud-config.yml
#cloud-config

ssh_authorized_keys:
  - ssh-rsa AAAAB3xxx
```

### waitjobコマンド

idcf-compute-apiとjqを便利に使うため、スクリプトを用意します。

``` bash ~/bin/waitjob
#!/usr/bin/env bash
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
    break;
  fi
done
```


### 作業マシンでCoreOSインストールの準備

IDCFクラウド上に適当なインスタンスを用意して、[idcf-compute-api](http://www.idcf.jp/cloud/docs/Getting%20Started)コマンドが利用できるようにします。

まず、[listDiskOfferings](http://www.idcf.jp/cloud/docs/api/user/listDiskOfferings)コマンドで、作成するディスクのタイプを確認します。

```
$  idcf-compute-api listDiskOfferings   | jq '.listdiskofferingsresponse.diskoffering[0] | {name,id}'
{
  "id": 10,
  "name": "Custom Disk"
}
```

[createVolume](http://www.idcf.jp/cloud/docs/api/user/createVolume)コマンドで、CoreOSをインストールするための空のディスクを作成します。

* diskofferingid : 10 (ディスクのタイプ)
* size: 40 (40GB)

``` bash
$ waitjob $(idcf-compute-api createVolume \
  --name=coreos_cluster_disk \
  --diskofferingid=10 \
  --size=40 \
  --zoneid=1 | jq '.createvolumeresponse.jobid')


{
  "jobresult": {
    "volume": {
...
      "type": "DATADISK",
      "diskofferingname": "Custom Disk",
      "diskofferingid": 10,
...
      "id": 80921,
      "isextractable": false,
      "name": "coreos_cluster_disk",
      "size": 42949672960,
      "state": "Allocated",
      "storage": "none",
      "storagetype": "shared"
    }
  },
  "jobid": 1131657
}
```

作業マシンのインスタンスのIDを[listVirtualMachines](http://www.idcf.jp/cloud/docs/api/user/listVirtualMachines)コマンドで確認しておきます。
クエリの方法はいろいろありますが、今回は表示名を使ってみます。

``` bash
$ idcf-compute-api listVirtualMachines | jq '.listvirtualmachinesresponse.virtualmachine[] | select(.displayname == "my-workstation").id'
53553
```

[attachVolume](http://www.idcf.jp/cloud/docs/api/user/attachVolume)コマンドで、作成したディスクを作業マシンにアタッチします。

* id: 80921 (作成したディスクのID)
* virtualmachineid: 53553 (作業マシンのID)

``` bash
$ waitjob $(idcf-compute-api attachVolume \
  --id=80921 \
  --virtualmachineid=53553 | jq '.attachvolumeresponse.jobid')
{
  "jobresult": {
    "volume": {
...
      "diskofferingid": 10,
      "diskofferingdisplaytext": "Custom Disk",
      "id": 80921,
...
      "name": "coreos_cluster_disk",
      "size": 42949672960,
      "state": "Ready"
    }
  },
  "jobid": 1131658
}
```

`fdisk`でアタッチされたディスクを確認すると、/dev/sddです。

``` bash
$ sudo fdisk -l | grep Disk

警告: GPT (GUID パーティションテーブル) が '/dev/sdb' に検出されました! この fdisk ユーティリティは GPT をサポートしません。GNU Parted を使ってください。

ディスク /dev/sdc は正常なパーティションテーブルを含んでいません
ディスク /dev/sdd は正常なパーティションテーブルを含んでいません
Disk /dev/sda: 85.9 GB, 85899345920 bytes
Disk /dev/sdb: 107.4 GB, 107374182400 bytes
Disk /dev/sdc: 42.9 GB, 42949672960 bytes
Disk /dev/sdd: 42.9 GB, 42949672960 bytes
```


### CoreOSのインストールスクリプト

CoreOSのインストールスクリプトをダウンロードします。

```
$ mkdir -p ~/coreos_apps/cluster
$ cd !$
$ wget https://raw.githubusercontent.com/coreos/init/master/bin/coreos-install
$ chmod u+x coreos-install
```

### クラスタ構成のcloud-config.yml

etcdクラスタを構成するため、各ノードがお互いのサービスディスカバリが必要になります。
前回はetcdクラスタの構成情報を保存する場所として、`discovery.etcd.io`の公開サービスを使いました。

discoveryAPIを使うため、最初にクラスタを識別するトークンを作成しておきます。

``` bash
$ curl https://discovery.etcd.io/new
https://discovery.etcd.io/217d781a5ad5c7c4660bfbbc239bd0c0
```

CoreOSはパスワード認証でSSH接続ができないため、ssh_authorized_keysに接続用の公開鍵を記述します。

``` yml  ~/coreos_apps/cluster/cloud-config.yml
#cloud-config
 
coreos:
  etcd:
      discovery: https://discovery.etcd.io/217d781a5ad5c7c4660bfbbc239bd0c0
      addr: $private_ipv4:4001
      peer-addr: $private_ipv4:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start

ssh_authorized_keys:
  - ssh-rsa AAAAB3xxx
```

### CoreOSのインストール

`coreos-install`コマンドを使い、マウントしたディスクにCoreOSをインストールします。
CoreOSのイメージはBetaチャネルを使います。

``` bash
$ sudo ./coreos-install -d /dev/sdd -C beta -c ./cloud-config.yml
Downloading and verifying coreos_production_image.bin.bz2...
--2014-06-28 19:24:49--  http://beta.release.core-os.net/amd64-usr/current/coreos_production_image.bin.bz2
...
Writing coreos_production_image.bin.bz2 to /dev/sdd...
  /tmp/coreos-install.IplQLdie0x/coreos_production_image.bin.bz2: done
Installing cloud-config...
Success! CoreOS beta current is installed on /dev/sdd
```

CoreOSをインストールしたディスクを作業マシンにmountして中身を確認します。

``` bash
$ sudo mkdir -p /coremnt
$ sudo mount -o subvol=root /dev/sdd9 /coremnt/
$ ls -al /coremnt/
```

`/var/lib/coreos-install/user_data`にインストールされているcloud-configを確認します。

``` bash
$ sudo cat /coremnt/var/lib/coreos-install/user_data
#cloud-config

coreos:
  etcd:
      discovery: https://discovery.etcd.io/217d781a5ad5c7c4660bfbbc239bd0c0
      addr: $private_ipv4:4001
      peer-addr: $private_ipv4:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start

ssh_authorized_keys:
  - ssh-rsa AAAAB3xxx
```

### テンプレートの作成

mountしたCoreOSのディスクをumountします。
デタッチしたディスクから直接テンプレートを作成できないので、一度スナップショットを作成します。手順が面倒です。

```
$ sudo umount /coremnt
```

[detachVolume](http://www.idcf.jp/cloud/docs/api/user/detachVolume)コマンドで、ディスクを作業マシンからデタッチします。

``` bash
$ waitjob $(idcf-compute-api detachVolume \
  --id=80921  | jq '.detachvolumeresponse.jobid')

{
  "jobresult": {
    "volume": {
...
      "id": 80921,
      "name": "coreos_cluster_disk",
      "size": 42949672960,
      "state": "Ready",
    }
  },
  "jobid": 1131668
}
```

[createSnapshot](http://www.idcf.jp/cloud/docs/api/user/createSnapshot)コマンドを使い、スナップショットの取得をします。今回は9分30秒かかりました。

* volumeid:80921 (デタッチしたボリュームID) 

``` bash
$ waitjob $(idcf-compute-api createSnapshot \
  --volumeid=80921 \
  | jq '.createsnapshotresponse.jobid')
........................................................

{
  "jobresult": {
    "snapshot": {
      "volumetype": "DATADISK",
      "volumename": "coreos_cluster_disk",
      "volumeid": 80921,
      "state": "CreatedOnPrimary",
...
      "id": 157309,
      "intervaltype": "MANUAL",
      "name": "detached_coreos_cluster_disk_20140628110329",
      "snapshottype": "MANUAL"
    }
  },
  "jobid": 1133135
}
```

テンプレート作成に必要なOSタイプを確認します。今回は"Other Linux (64-bit)"を指定します。

``` bash
$ idcf-compute-api listOsTypes | jq '.listostypesresponse.ostype[] | select(contains({description:"Other Linux (64-bit)"}))'
{
  "oscategoryid": 7,
  "id": 99,
  "description": "Other Linux (64-bit)"
}

```

[createTemplate](http://www.idcf.jp/cloud/docs/api/user/createTemplate)コマンドを使い、スナップショットからテンプレートを作成します。

* snapshotid: 157309 (作成したスナップショットID)

``` bash
$ waitjob $(idcf-compute-api createTemplate \
  --snapshotid=157309 \
  --displaytext=coreos-cluster-tpl-v2 \
  --name=coreos-cluster-tpl-v2 \
  --ostypeid=99 | jq '.createtemplateresponse.jobid')
..

{
  "jobresult": {
    "template": {
...
      "status": "Download Complete",
      "size": 42949672960,
      "passwordenabled": false,
      "ostypename": "Other Linux (64-bit)",
      "format": "OVA",
...

      "id": 8599,
      "isextractable": false,
      "isfeatured": false,
      "ispublic": false,
      "isready": true,
      "name": "coreos-cluster-tpl-v2",
      "ostypeid": 103
    }
  },
  "jobid": 1133136
}
```

[listTemplates](http://www.idcf.jp/cloud/docs/api/user/listTemplates)コマンドで、作成したテンプレートを確認します。

* id: 8599 (作成したテンプレートID)

``` bash
$ idcf-compute-api listTemplates \
  --id=8599 \
  --templatefilter=self \
  | jq '.listtemplatesresponse.template[] | {id,name,size,isready}'
{
  "isready": true,
  "size": 42949672960,
  "name": "coreos-cluster-tpl-v2",
  "id": 8599
}
```

### インスタンスの作成は失敗

[deployVirtualMachine](http://www.idcf.jp/cloud/docs/api/user/deployVirtualMachine)は失敗してしまいました。。。

原因はまだわからないので、デバッグしていきます。なかなかうまく行かないものです。

``` bash
$ waitjob $(idcf-compute-api deployVirtualMachine \
  --serviceofferingid=21 \
  --templateid=8599 \
  --zoneid=1 \
  --name=coreos-cluster-v2 \
  --displayname=coreos-cluster-v2 \
  --group=mshimizu | jq '.deployvirtualmachineresponse.jobid')
....

{
  "jobresult": {
    "errortext": "Unable to create a deployment for VM[User|xxx]",
    "errorcode": 533
  },
  "jobid": 1133137
}
```

### まとめ

今回はCoreOSのクラスタ作成用のテンプレートを作成しました。

CoreOSを直接ディスクにインストールしているため、テンプレートにuserdataにdiscoveryAPI用のTOKENが埋め込まれてしまっています。そのため、このテンプレートから作成するインスタンスはすべて同じクラスタ構成になってしまいます。

とりあえずテスト用なのでこのままCoreOSクラスタを構築しますが、別のクラスタを作る場合はTOKENを変更するのが面倒です。
起動後に変更できたとしても、一度古いTOKENでクラスタのノードを見てしまうので誤動作そうです。

そもそもまだインスタンスの起動に成功していないのですが。

理想はGCEのように、インスタンスの起動時にcloud-initでuserdataに指定したcloud-configを実行できると便利です。

IDCFクラウドのCloudStackが古いため、cloud-initで使っているCloudStackAPIが使えるかどうか確認する必要があります。
userdataを使い、インスタンス起動時に実行したいスクリプトや環境変数を渡せるので、いろいろ便利に使えそう。




