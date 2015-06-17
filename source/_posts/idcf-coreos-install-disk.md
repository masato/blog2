title: 'IDCFクラウドでCoreOSをディスクからインストールする'
date: 2014-06-03 22:47:22
tags:
 - IDCFクラウド
 - CoreOS
 - GCE
 - cloud-config
 - idcf-compute-api
 - Vultr
 - ProjectAtomic
description: Dockerコンテナのプロダクション運用を考えると、2014-05-23にCoreOSの正式サポートを発表したGCE、OrchardやStackdockのDocker-as-a-Service、DigitalOceanやVultrなどのVPSが候補になります。特にGCEはインスタンス起動がとても速く、gcutilでuser-dataにcloud-configを渡せるのでCoreOSの作成がとても簡単です。何回かIDCFクラウドでもCoreOSを使いたくて、ディスクやISOへのインストールを試しましたが、失敗してしまいました。BeagleBoneBlackへUbuntuをインストールしている間に、ディスクにインストールする方法がわかってきたので、また試してみます。
---

* `Update 2014-06-28`: [IDCFクラウドにCoreOSクラスタを構築する - Part1: CoreOSの準備](/2014/06/28/idcf-coreos-cluster)
* `Update 2014-07-10`: [IDCFクラウドにCoreOSクラスタを構築する - Part2: CoreOSのディスカバリ](/2014/07/10/idcf-coreos-cluster-discovery)
* `Update 2014-07-15`: [IDCFクラウドにCoreOSクラスタを構築する - Part3: クラスタをセットアップ](/2014/07/15/idcf-coreos-cluster-setting-up)
* `Update 2014-07-17`: [IDCFクラウドにCoreOSクラスタを構築する - Part4: 367.1.0のissue](/2014/07/17/idcf-coreos-cluster-36710/)
* `Update 2014-07-19`: [IDCFクラウドにCoreOSクラスタを構築する - Part5: 再セットアップ](/2014/07/19/idcf-coreos-cluster-re-setting-up/)
* `Update 2014-08-18`: [IDCFクラウドでCoreOSをISOからインストールする](/2014/08/15/deis-in-idcf-cloud-deploy/)

Dockerコンテナのプロダクション運用を考えると、2014-05-23にCoreOSの正式サポートを[発表](http://googlecloudplatform.blogspot.jp/2014/05/official-coreos-images-are-now-available-on-google-compute-engine.html)したGCE、
[Orchard](https://orchardup.com/)や[Stackdock](https://stackdock.com/)の`Docker-as-a-Service`、DigitalOceanや[Vultr](https://www.vultr.com/)などのVPSが候補になります。

特に[GCE](/2014/05/12/gce-coreos-cluster-prepare/)はインスタンス起動がとても速く、gcutilでuser-dataにcloud-configを渡せるのでCoreOSの作成がとても簡単です。

何回かIDCFクラウドでもCoreOSを使いたくて、[ディスク](https://coreos.com/docs/running-coreos/bare-metal/installing-to-disk/)や[ISO](https://coreos.com/docs/running-coreos/platforms/iso/)へのインストールを試しましたが、失敗してしまいました。
`BeagleBoneBlack`へ[Ubuntuをインストール](/2014/05/03/beagleboneblack-ubuntu/)している間に、ディスクにインストールする方法がわかってきたので、また試してみます。


### TL;DR
IDCFクラウド上でもCoreOSが動きました！VagrantやGCEで試していたクラスタ構築も次回試します。

<!-- more -->

### はじめに

ディスクへの[インストール手順](https://coreos.com/docs/running-coreos/bare-metal/installing-to-disk/)はいつのまにか更新されていました。
ただ、CoreOSのドキュメントは全体的にちょっとインデックスがわかりづらいです。


[idcf-compute-api](http://www.idcf.jp/cloud/docs/)を便利に使うために、非同期ジョブの終了をポーリングして待機するスクリプトを用意しました。

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

実行権限をつけておきます。

``` bash
$ chmod u+x ~/bin/waitjob
```

### ディスクの準備

作成するディスクの[タイプ](http://www.idcf.jp/cloud/docs/api/user/listDiskOfferings)を確認します。

``` bash
$ idcf-compute-api listDiskOfferings -t=id,name
+----+-------------+
| id | name        |
+----+-------------+
| 10 | Custom Disk |
+----+-------------+
```

80GBの[ディスクを作成](http://www.idcf.jp/cloud/docs/api/user/createVolume)します。

``` bash
$ waitjob $(idcf-compute-api createVolume \
  --name=coreos_disk \
  --diskofferingid=10 \
  --size=80 \
  --zoneid=1 | jq '.createvolumeresponse | .jobid' )
{
  "jobid": 1084505
}
{
  "jobresult": {
    "volume": {
...
      "diskofferingname": "Custom Disk",
      "diskofferingid": 10,
...
      "name": "coreos_disk",
      "size": 85899345920,
      "state": "Allocated",
...
```

作成したディスクを作業マシンに[アタッチ](http://www.idcf.jp/cloud/docs/api/user/attachVolume)します。

``` bash
$ waitjob $(idcf-compute-api attachVolume \
  --id=xxx \
  --virtualmachineid=xxx | jq '.attachvolumeresponse | .jobid')
{
  "jobid": 1084507
}
{
  "jobresult": {
    "volume": {
...
      "vmstate": "Running",
...
      "type": "DATADISK",
...
      "diskofferingname": "Custom Disk",
      "diskofferingid": 10,
...
      "name": "coreos_disk",
      "size": 85899345920,
      "state": "Ready"
...
```

一度作業マシンをrebootします。

``` bash
$ sodo reboot
```

reboot後に、fdiskでアタッチしたディスクを確認します。80GBが2つありますが、/dev/sdaはブートディスクなので
今回追加したディスクは、/dev/sddになります。

``` bash
$ sudo fdisk -l | grep Disk
....
Disk /dev/sda: 85.9 GB, 85899345920 bytes
Disk /dev/sdb: 107.4 GB, 107374182400 bytes
Disk /dev/sdc: 42.9 GB, 42949672960 bytes
Disk /dev/sdd: 85.9 GB, 85899345920 bytes
```

### インストールスクリプトの実行

CoreOSの[インストールスクリプト](https://raw.github.com/coreos/init/master/bin/coreos-install)をダウンロードします。

``` bash
$ cd ~/coreos_apps/
$ wget https://raw.githubusercontent.com/coreos/init/master/bin/coreos-install
$ chmod u+x coreos-install
```

cloud-configを用意します。
CoreOSはパスワード認証でSSH接続ができないため、ssh_authorized_keysに接続用の公開鍵を記述します。

``` bash ~/coreos_apps/config
#cloud-config

ssh_authorized_keys:
  - ssh-rsa xxx
```

coreos-installを実行します。-cオプションで先ほど作成したcloud-configを指定します。

``` bash
$ sudo ./coreos-install -d /dev/sdd -C beta -c ./config
Downloading and verifying coreos_production_image.bin.bz2...
...
Writing coreos_production_image.bin.bz2 to /dev/sdd...
  /tmp/coreos-install.wMnXNZkXfg/coreos_production_image.bin.bz2: done
Installing cloud-config...
Success! CoreOS beta current is installed on /dev/sdd
```

CoreOSをインストールしたディスクを作業マシンにmountして中身を確認します。

``` bash
$ sudo mkdir /coremnt
$ sudo mount -o subvol=root /dev/sdd9 /coremnt/
$ ls -al /coremnt/
```

cloud-configは、`/var/lib/coreos-install/user_data`にインストールされています。

``` bash
$ sudo cat /coremnt/var/lib/coreos-install/user_data
#cloud-config

ssh_authorized_keys:
  - ssh-rsa　xxx
```

### テンプレートの作成

ディスクを作業マシンからデタッチして、一度スナップショットを作成してからテンプレートを作成します。

確認のためmountしたCoreOSのディスクをumountします。

``` bash
$ sudo umount /coremnt
```

ディスクを作業マシンから[デタッチ](http://www.idcf.jp/cloud/docs/api/user/detachVolume)します。
``` bash
$ waitjob $(idcf-compute-api detachVolume \
  --id=xxx | jq '.detachvolumeresponse | .jobid')
```

[スナップショットの取得](http://www.idcf.jp/cloud/docs/api/user/createSnapshot)をします。

``` bash
$ waitjob $(idcf-compute-api createSnapshot \
  --volumeid=xxx | jq '.createsnapshotresponse | .jobid')
```

事前にテンプレートを作成するときの[OSタイプ](http://www.idcf.jp/cloud/docs/api/user/listOsTypes)を確認します。
何でもよいですが、`Other (64-bit)`を使うことにします。

``` bash
$ idcf-compute-api listOsTypes -c | grep 'Other Linux (64-bit)'
"Other Linux (64-bit)",99,7
```

最後に、[テンプレートの作成](http://www.idcf.jp/cloud/docs/api/user/createTemplate)をします。

``` bash
$ waitjob $(idcf-compute-api createTemplate \
  --snapshotid=xxx \
  --displaytext=coreos-tpl-v2 \
  --name=coreos-tpl-v2 \
  --ostypeid=99 | jq '.createtemplateresponse  | .jobid')
```

### インスタンスの作成

予め作成するインスタンスの[タイプを確認](http://www.idcf.jp/cloud/docs/api/user/listServiceOfferings)します。
今回はM4 ( 2CPU / 4GB )で作成します。

``` bash
$ idcf-compute-api listServiceOfferings | jq '.listserviceofferingsresponse | .serviceoffering[] | select(.name == "M4") | {id,displaytext}'
{
  "displaytext": "M4 ( 2CPU / 4GB )",
  "id": 21
}
```

[インスタンスの作成](http://www.idcf.jp/cloud/docs/api/user/deployVirtualMachine)をします。

``` bash
$ waitjob $(idcf-compute-api deployVirtualMachine \
  --serviceofferingid=21 \
  --templateid=xxx \
  --zoneid=1 \
  --name=coreos-v2 \
  --displayname=coreos-v2 \
  --group=mshimizu | jq '.deployvirtualmachineresponse | .jobid')
```

### SSHでログインして確認

作業マシンからSSHで接続します。CoreOSのユーザー名は`core`です。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/private_key
$ ssh -A core@coreos-v2
CoreOS (beta)
core@coreos-v2 ~ $
```

dockerとfleetのバージョンを確認します。

``` bash
$ docker -v
Docker version 0.11.1, build fb99f99
$ fleet -version
fleet version 0.3.3
```

### まとめ

RedHatが[Project Atomic](http://www.projectatomic.io/)を発表して、軽量OSとコンテナクラスタの分散管理は非常におもしろくなっています。
`Datacenter-as-a-Computer`を目指そうとすると、ChromeOSを参考にしているCoreOSの方が楽しそうです。
DartやGoも勉強中なので。



