title: "Deis in IDCFクラウド - Part1: CoreOSクラスタとDeisインストール"
date: 2014-08-14 22:57:51
tags:
 - Deis
 - CoreOS
 - IDCFクラウド
 - GCE
description: IDCFクラウド向けにはセットアップツールは用意されていないのでProvision a Deis Cluster on bare-metal hardwareとDeis in Google Compute Engineを読みながら参考にしていきます。Deisのバージョンは、v0.10.0を使います。GCEのサンプルはChefを使わない手動インストールなので、手順が透過的でわかりやすいです。やはりIaaSにロードバランサとDNSサービスが提供されていてAPIで操作できるのは便利です。特にDockerの場合は顕著です。残念ながらIDCFクラウドにはどちらもないので自分で解決していきます。DeisもCoreOSを使っているので、参考にするのはベアメタルのセットアップドキュメントになります。面倒ですが汎用的なセットアップ手順を学べるので良いことだと思うことにします。VagrantやChefでかんたん！という風潮もちょっとどうかと。
---

* `Update 2014-08-15`: [Deis in IDCFクラウド - Part2: SinatraをDeisにデプロイ](/2014/08/15/deis-in-idcf-cloud-deploy/)
* `Update 2014-08-18`: [IDCFクラウドでCoreOSをISOからインストールする](/2014/08/15/deis-in-idcf-cloud-deploy/)

/2014/08/18/idcf-coreos-install-from-iso/

IDCFクラウド向けにはセットアップツールは用意されていないので[Provision a Deis Cluster on bare-metal hardware](https://github.com/deis/deis/tree/master/contrib/bare-metal)と[Deis in Google Compute Engine](https://gist.github.com/andyshinn/a78b617b2b16a2782655)を読みながら参考にしていきます。
Deisのバージョンは、`v0.10.0`を使います。

GCEのサンプルはChefを使わない手動インストールなので、手順が透過的でわかりやすいです。
やはりIaaSにロードバランサとDNSサービスが提供されていてAPIで操作できるのは便利です。
特にDockerの場合は顕著です。残念ながらIDCFクラウドにはどちらもないので自分で解決していきます。

DeisもCoreOSを使っているので、参考にするのはベアメタルのセットアップドキュメントになります。
面倒ですが汎用的なセットアップ手順を学べるので良いことだと思うことにします。VagrantやChefでかんたん！という風潮もちょっとどうかと。

<!-- more -->

### CoreOSクラスタの構築

CoreOSのインスタンスは3台作成します。[IDCFクラウドにCoreOSクラスタを構築する - Part5: 再セットアップ](/2014/07/19/idcf-coreos-cluster-re-setting-up/)のCoreOSはBetaチャンネルであったのと、cloud-config.ymlもDeis用の設定が必要なので、同じ手順で再作成します。


プロジェクトを作成します。

``` bash ~/coreos_apps/deis
$ mkdir deis
$ cp cloud-config.yml deis/
$ cd deis/
```

### Deis用のcloud-config.yml

[user-data](https://github.com/deis/deis/blob/master/contrib/coreos/user-data)を参考にして、Deis用にカスタムしたcloud-configを作成します。

* `/etc/environment`は、明示的にwrite_filesで作成する。
* etcdからイメージ名を取得するため、`/run/deis/bin/get_image`の実行ファイルを作成する。

```  ~/coreos_apps/deis/cloud-config.yml
#cloud-config

ssh_authorized_keys:
    - ssh-rsa AAAAB3NzaC1yc...

coreos:
    etcd:
        # generate a new token for each unique cluster from https://discovery.etcd.io/new
        discovery: https://discovery.etcd.io/87cb67ed880353c066d5a0dd63331a31
        # multi-region and multi-cloud deployments need to use $public_ipv4
        addr: $private_ipv4:4001
        peer-addr: $private_ipv4:7001
    fleet:
        # We have to set the public_ip here so this works on Vagrant -- otherwise, Vagrant VMs
        # will all publish the same private IP. This is harmless for cloud providers.
        public_ip: $private_ipv4
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
        COREOS_PUBLIC_IPV4=$public_ipv4
        COREOS_PRIVATE_IPV4=$private_ipv4
  - path: /etc/deis-release
    content: |
      DEIS_RELEASE=latest
  - path: /etc/motd
    content: " \e[31m* *    \e[34m*   \e[32m*****    \e[39mddddd   eeeeeee iiiiiii   ssss\n\e[31m*   *  \e[34m* *  \e[32m*   *     \e[39md   d   e    e    i     s    s\n \e[31m* *  \e[34m***** \e[32m*****     \e[39md    d  e         i    s\n\e[32m*****  \e[31m* *    \e[34m*       \e[39md     d e         i     s\n\e[32m*   * \e[31m*   *  \e[34m* *      \e[39md     d eee       i      sss\n\e[32m*****  \e[31m* *  \e[34m*****     \e[39md     d e         i         s\n  \e[34m*   \e[32m*****  \e[31m* *      \e[39md    d  e         i          s\n \e[34m* *  \e[32m*   * \e[31m*   *     \e[39md   d   e    e    i    s    s\n\e[34m***** \e[32m*****  \e[31m* *     \e[39mddddd   eeeeeee iiiiiii  ssss\n\n\e[39mWelcome to Deis\t\t\tPowered by Core\e[38;5;45mO\e[38;5;206mS\e[39m\n"
  - path: /etc/profile.d/nse-function.sh
    permissions: 0755
    content: |
      function nse() {
        sudo nsenter --pid --uts --mount --ipc --net --target $(docker inspect --format="&#123;&#123; .State.Pid &#125;&#125;" $1)
      }
  - path: /run/deis/bin/get_image
    permissions: 0755
    content: |
      #!/bin/bash
      # usage: get_image <component_path>
      IMAGE=`etcdctl get $1/image 2>/dev/null`

      # if no image was set in etcd, we use the default plus the release string
      if [ $? -ne 0 ]; then
        RELEASE=`etcdctl get /deis/release 2>/dev/null`

        # if no release was set in etcd, use the default provisioned with the server
        if [ $? -ne 0 ]; then
          source /etc/deis-release
          RELEASE=$DEIS_RELEASE
        fi

        IMAGE=$1:$RELEASE
      fi
      # remove leading slash
      echo ${IMAGE#/}
```

### CoreOSのセットアップおさらい

CoreOSをディスクにインストールするため、新しいボリュームを作成します。

```
$ waitjob $(idcf-compute-api createVolume \
  --name=coreos_alpha \
  --diskofferingid=10 \
  --size=80 \
  --zoneid=1 | jq '.createvolumeresponse.jobid' )
```

ボリュームを作業マシンにアタッチします。

```
$ waitjob $(idcf-compute-api attachVolume \
  --id=85607 \
  --virtualmachineid=53553 | jq '.attachvolumeresponse.jobid')
```


アタッチされたボリュームは、/dev/sdcにマウントされます。
alphaチャンネルを指定して、ディスクにCoreOSをインストールします。

```
$ sudo ./coreos-install -C alpha -d /dev/sdc -c ./cloud-config.yml
```

ボリュームをデタッチします。

```
$ waitjob $(idcf-compute-api detachVolume \
  --id=85607 | jq '.detachvolumeresponse.jobid')
```

スナップショットを作成します。

```
$ waitjob $(idcf-compute-api createSnapshot \
  --volumeid=85607 \
  | jq '.createsnapshotresponse.jobid')
```

OVAテンプレートを作成します。

```
$ waitjob $(idcf-compute-api createTemplate \
  --snapshotid=174692\
  --displaytext=coreos-alpha-tpl-v12 \
  --name=coreos-alpha-tpl-v12 \
  --ostypeid=99 | jq '.createtemplateresponse.jobid')
```

`--name`と`--displayname`を変更してインスタンスを3台作成します。

```
$ waitjob $(idcf-compute-api deployVirtualMachine \
  --serviceofferingid=21 \
  --templateid=9397\
  --zoneid=1 \
  --name=coreos-alpha-v12-1 \
  --displayname=coreos-alpha-v12-1 \
  --group=mshimizu | jq '.deployvirtualmachineresponse.jobid')
```

### CoreOSの設定変更

CoreOSのSSH接続して、いくつか設定変更後にrebootします。cloud-configで設定した`/etc/motd`がかわいい。

```
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh -A core@10.1.0.249
 * *    *   *****    ddddd   eeeeeee iiiiiii   ssss
*   *  * *  *   *     d   d   e    e    i     s    s
 * *  ***** *****     d    d  e         i    s
*****  * *    *       d     d e         i     s
*   * *   *  * *      d     d eee       i      sss
*****  * *  *****     d     d e         i         s
  *   *****  * *      d    d  e         i          s
 * *  *   * *   *     d   d   e    e    i    s    s
***** *****  * *     ddddd   eeeeeee iiiiiii  ssss

Welcome to Deis                 Powered by CoreOS
core@core3 ~ $
```

`/etc/environmnet`の`COREOS_PRIVATE_IPV4`にアサインされたIPアドレスを指定します。

``` bash /etc/environmnet
COREOS_PUBLIC_IPV4=
COREOS_PRIVATE_IPV4=10.1.0.249
```

cloud-configのhostnameをFQDNで指定します。

``` yaml /var/lib/coreos-install/user_data
#cloud-config
hostname: core3.10.1.0.249.xip.io
```

CoreOSをrebootします。

```
$ sudo reboot
```

再度SSH接続して起動を確認します。

```
core@core3 ~ $ hostname
core3.10.1.0.249.xip.io
```

同様に3台すべてを設定変更をしてrebootします。
fleetctlでCoreOSクラスタのマシンを確認します。

```
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ export FLEETCTL_TUNNEL=10.1.2.34
$ fleetctl list-machines
MACHINE         IP              METADATA
a34af2af...     10.1.2.34       -
b142e819...     10.1.3.33       -
eece00ef...     10.1.0.249      -
```


### 作業マシンにfleetctlをインストール

Deisのインストールはfleetctlを使うため、作業マシンにCoreOSと同じバージョンのfleetctlをインストールします。
同じ`v0.6.2`でも、`git clone`したfleetctlを使うとバージョンチェックでエラーになります。

```
Your fleetctl client version should match the server. Local version: fleetctl version 0.6.2+git, server version: fleetctl version 0.6.2. Install the appropriate version from https://github.com/coreos/fleet/releases
```

fleetctlのバイナリをダウンロードして作業マシンにインストールします。

```
$ cd ~/Downloads
$ wget https://github.com/coreos/fleet/releases/download/v0.6.2/fleet-v0.6.2-linux-amd64.tar.gz
$ tar zxvf fleet-v0.6.2-linux-amd64.tar.gz
$ sudo mv fleet-v0.6.2-linux-amd64/fleet* /usr/local/bin
```

`FLEETCTL_TUNNEL`の環境変数にCoreOSクラスタのうち1台を指定します。
作業マシンからfleetctlが使えることを確認しました。

``` 
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ export FLEETCTL_TUNNEL=10.1.2.34
$ fleetctl version
fleetctl version 0.6.2
$ fleetctl list-machines
MACHINE         IP              METADATA
a34af2af...     10.1.2.34       -
b142e819...     10.1.3.33       -
eece00ef...     10.1.0.249      -
```

### Deisのインストール

[deis](https://github.com/deis/deis.git)をGitHubからCloneします。
Deisの各コンポーネントはCoreOSのUnitとしてインストールされます。Unit間の連携や、CoreOSクラスタの使い方の参考にもなります。
Makefileの`make run`を実行してしばらくするとインストールが完了します。

```
$ git clone https://github.com/deis/deis.git
$ cd deis
$ make run
...
Waiting for deis-builder to start...
fleetctl --strict-host-key-checking=false start -no-block builder/systemd/*.service
Triggered job deis-builder.service start
Your Deis cluster is ready to go! Continue following the README to login and use Deis.
```

アンインストールしたい場合は、`make uninstall`でUnitのstopとdestroyを行ってくれます。

`fleetctl list-units`を実行してCoreOSクラスタにインストールされたDeisのUnitsを確認します。

```
$ fleetctl list-units
UNIT                            DSTATE          TMACHINE                STATE  MACHINE                  ACTIVE
deis-builder-data.service       loaded          eece00ef.../10.1.0.249  loaded eece00ef.../10.1.0.249   active
deis-builder.service            launched        eece00ef.../10.1.0.249  launchedeece00ef.../10.1.0.249  active
deis-cache.service              launched        b142e819.../10.1.3.33   launchedb142e819.../10.1.3.33   active
deis-controller.service         launched        eece00ef.../10.1.0.249  launchedeece00ef.../10.1.0.249  active
deis-database-data.service      loaded          eece00ef.../10.1.0.249  loaded eece00ef.../10.1.0.249   active
deis-database.service           launched        eece00ef.../10.1.0.249  launchedeece00ef.../10.1.0.249  active
deis-logger-data.service        loaded          eece00ef.../10.1.0.249  loaded eece00ef.../10.1.0.249   active
deis-logger.service             launched        eece00ef.../10.1.0.249  launchedeece00ef.../10.1.0.249  active
deis-registry-data.service      loaded          eece00ef.../10.1.0.249  loaded eece00ef.../10.1.0.249   active
deis-registry.service           launched        eece00ef.../10.1.0.249  launchedeece00ef.../10.1.0.249  active
deis-router@1.service           launched        eece00ef.../10.1.0.249  launchedeece00ef.../10.1.0.249  active
```

### まとめ

Deisのインストールは終了しましたが、このままではまだ使えません。
次回はロードバランサとDNSの設定をして、Deisにサンプルアプリをデプロイするところまでやりたいです。

GoogleだとロードバランサやDNSがAPI経由でできて便利なのですが。。