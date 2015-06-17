title: 'IDCFクラウドにCoreOSクラスタを構築する - Part5: 再セットアップ'
date: 2014-07-19 20:35:42
tags:
 - IDCFクラウド
 - CoreOS
 - jq
 - まとめ
description: IDCFクラウドにCoreOSクラスタを構築する - Part3 セットアップのやり直しです。CoreOSは絶賛開発中なのでキャッチアップしていかないとすぐに使えなくなります。はやくDeisとKubernetesをインストールする環境を用意したいです。
---

* `Update 2014-08-18`: [IDCFクラウドでCoreOSをISOからインストールする](/2014/08/15/deis-in-idcf-cloud-deploy/)

### まとめ

* [IDCFクラウドにCoreOSクラスタを構築する - Part1: CoreOSの準備](/2014/06/28/idcf-coreos-cluster)
* [IDCFクラウドにCoreOSクラスタを構築する - Part2: CoreOSのディスカバリ](/2014/07/10/idcf-coreos-cluster-discovery)
* [IDCFクラウドにCoreOSクラスタを構築する - Part3: クラスタをセットアップ](/2014/07/15/idcf-coreos-cluster-setting-up)
* [IDCFクラウドにCoreOSクラスタを構築する - Part4: 367.1.0のissue](/2014/07/17/idcf-coreos-cluster-36710/)
* [IDCFクラウドにCoreOSクラスタを構築する - Part5: 再セットアップ](/2014/07/19/idcf-coreos-cluster-re-setting-up/)


[Part3](/2014/07/15/idcf-coreos-cluster-setting-up/)のやり直しです。CoreOSは絶賛開発中なのでキャッチアップしていかないとすぐに使えなくなります。
はやくDeisとKubernetesをインストールする環境を用意したいです。


<!-- more -->

### テンプレートの作成

[ディスクへインストール](/2014/06/03/idcf-coreos-install-disk/)の手順でテンプレートを作成しておきます。今回は復習です。

最初に`~/bin/waitjob`を用意します。

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

ボリュームの作成とボリュームのアタッチをします。

``` bash 
$ waitjob $(idcf-compute-api createVolume \
  --name=coreos_disk \
  --diskofferingid=10 \
  --size=80 \
  --zoneid=1 | jq '.createvolumeresponse.jobid' )
  
$ waitjob $(idcf-compute-api attachVolume \
  --id=83223 \
  --virtualmachineid=53553 | jq '.attachvolumeresponse.jobid')
```

cloud-config.ymlを作成します。

``` yml ~/coreos_apps/cloud-config.yml

#cloud-config

ssh_authorized_keys:
    - ssh-rsa AAAAB3NzaC1yc2E....

write_files:
  - path: /etc/environment
    permissions: 0644
    content: |
        COREOS_PUBLIC_IPV4=$public_ipv4
        COREOS_PRIVATE_IPV4=$private_ipv4

coreos:
    etcd:
        discovery: https://discovery.etcd.io/ed724cf0abd2d02abf985f57b53d6cf4
        addr: $private_ipv4:4001
        peer-addr: $private_ipv4:7001
    units:
      - name: etcd.service
        command: start
      - name: fleet.service
        command: start
```

cloud-config.ymlを使いCoreOSをディスクにインストールします。

``` bash
$ sudo ./coreos-install -d /dev/sdc -c ./cloud-config.yml
```

ボリュームをデタッチ、スナップショットを作成してからテンプレートを作成します。

``` bash
$ waitjob $(idcf-compute-api detachVolume \
  --id=83223 | jq '.detachvolumeresponse.jobid')

$ waitjob $(idcf-compute-api createSnapshot \
  --volumeid=83223 \
  | jq '.createsnapshotresponse.jobid')

$ waitjob $(idcf-compute-api createTemplate \
  --snapshotid=166218\
  --displaytext=coreos-beta-tpl-v4 \
  --name=coreos-beta-tpl-v4 \
  --ostypeid=99 | jq '.createtemplateresponse.jobid')
```

インスタンスを作成します。

``` bash
$ waitjob $(idcf-compute-api deployVirtualMachine \
  --serviceofferingid=21 \
  --templateid=9030\
  --zoneid=1 \
  --name=coreos-beta-v3-3 \
  --displayname=coreos-beta-v3-3 \
  --group=mshimizu | jq '.deployvirtualmachineresponse.jobid')
```

### CoreOSインスタンスにログイン

公開鍵認証でSSHログインします。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/my_private_key
$ ssh -A core@coreos-beta-v3-3
CoreOS (beta)
```

ifconfigでIPアドレスを確認します。

``` bash
$ ifconfig ens32 | awk '/inet /{print $2}'
10.1.0.117
```

確認したIPアドレスを`/etc/environment`の`COREOS_PRIVATE_IPV4`に記述します。

``` bash 
$ sudo vi /etc/environment
COREOS_PUBLIC_IPV4=
COREOS_PRIVATE_IPV4=10.1.0.117
```

rebootします。

```
$ sudo reboot
```

reboot後、`fleetctl list-machines`でCoreOSクラスタのジョインしているインスタンスを確認します。

``` bash
$ fleetctl list-machines
MACHINE         IP              METADATA
0ba88b13...     10.1.0.174      -
3b9bf346...     10.1.1.90       -
5d2a8312...     10.1.0.117      -
```

### まとめ

CoreOSをディスクへのインストールは面倒です。[公式のISO](http://coreos.com/docs/running-coreos/platforms/iso/)が提供されているので今度はISOを使ってみようと思います。

discoveryのTOKENのリセットもできるようになったので、CoreOSクラスタのインストールは終了できそうです。
今度こそDeisのインストールができます。


