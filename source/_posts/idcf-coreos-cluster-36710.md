title: 'IDCFクラウドにCoreOSクラスタを構築する - Part4: 367.1.0のissue'
date: 2014-07-17 22:39:42
tags:
 - IDCFクラウド
 - CoreOS
description: IDCFクラウドにCoreOSクラスタを構築する - Part3 セットアップで成功したときのCoreOSのバージョンを記録していないのですが、またインストールに失敗するようになりました。今回はバージョンを確認して記録します。CoreOSの367.1.0をbeta channelでIDCFクラウドでディスクにインストールすると、discoveryに失敗してしまいます。
---

* `Update 2014-09-22`: [Kubernetes on CoreOS with Fleet and Rudder on IDCFクラウド - Part5: FleetとRudderで再作成する](/2014/09/22/kubernetes-fleet-rudder-on-idcf/)
* `Update 2014-07-19`: [IDCFクラウドにCoreOSクラスタを構築する - Part5: 再セットアップ](/2014/07/19/idcf-coreos-cluster-re-setting-up/)
* `Update 2014-08-18`: [IDCFクラウドでCoreOSをISOからインストールする](/2014/08/15/deis-in-idcf-cloud-deploy/)

[IDCFクラウドにCoreOSクラスタを構築する - Part3: セットアップ](/2014/07/15/idcf-coreos-cluster-setting-up/)で成功したときのCoreOSのバージョンを記録していないのですが、またインストールに失敗するようになりました。

今回はバージョンを確認して記録します。
CoreOSの367.1.0を`beta channel`でIDCFクラウドでディスクにインストールすると、discoveryに失敗してしまいます。

<!-- more -->


### CoreOS Stable Release

* `Updated 2014-08-18`: [CoreOS Stable Release](https://coreos.com/blog/stable-release/)
`367.1.0`は、`Stable Channel`としてリリースされました。
 
### CoreOSのバージョン確認

インスタンスはCoreOSの367.1.0を`beta channel`でインストールしています。

``` bash
$ cat /etc/os-release
NAME=CoreOS
ID=coreos
VERSION=367.1.0
VERSION_ID=367.1.0
BUILD_ID=
PRETTY_NAME="CoreOS 367.1.0"
ANSI_COLOR="1;32"
HOME_URL="https://coreos.com/"
BUG_REPORT_URL="https://github.com/coreos/bugs/issues"
```

### journalctlでログの確認

``` bash
$ journalctl -u etcd.service
...
CRITICAL  | failed sanitizing configuration: Advertised URL: parse :4001: missing protocol scheme
```

### systemctlでunitファイルの確認

systemctlでunitファイルを読むと、`ETCD_ADDR`と`ETCD_PEER_ADDR`がポートだけでURLが書かれていません。
367.1.0では、`COREOS_PRIVATE_IP`が入らないようです。

systemdのunitファイルは以下のディレクトリにあります。

`/usr/lib64/systemd/system/`

``` bash
$ systemctl cat etcd.service
# /usr/lib64/systemd/system/etcd.service
[Unit]
Description=etcd

[Service]
User=etcd
PermissionsStartOnly=true
Environment=ETCD_DATA_DIR=/var/lib/etcd ETCD_NAME=default
ExecStart=/usr/bin/etcd
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

# /run/systemd/system/etcd.service.d/20-cloudinit.conf
[Service]
Environment="ETCD_ADDR=:4001"
Environment="ETCD_DISCOVERY=https://discovery.etcd.io/ed724cf0abd2d02abf985f57b5
Environment="ETCD_NAME=75aa20edd64b40488cccd0b5f3d419d4"
Environment="ETCD_PEER_ADDR=:7001"
```

[missing /etc/environment file](https://github.com/coreos/bugs/issues/65)のissueを読むと、確かに`/etc/environment`ファイルが見つかりません。

``` bash
$ cat /usr/share/oem/bin/coreos-setup-environment
cat: /usr/share/oem/bin/coreos-setup-environment: No such file or directory
$ cat /etc/environment
cat: /etc/environment: No such file or directory
```

[No IP data in cloudinit on OpenStack](https://github.com/coreos/bugs/issues/67)でOpenStackでも同様のissueがあります。

> Same problem here, COREOS_PRIVATE_IP is empty using 367.1.0 on openstack with cloud-init.

/etc/environmentは明示的に作成する必要があるようです。

``` yaml
#cloud-config

write_files:
  - name: /etc/environment
    content: |
        COREOS_PUBLIC_IPV4=$public_ipv4
        COREOS_PRIVATE_IPV4=$private_ipv4
```

そういえば、[Installing CoreOS to Disk](http://coreos.com/docs/running-coreos/bare-metal/installing-to-disk/)に以下のような記述がありました。
なんのことかわからなかったのですが、どうもこのことらしいです。

> Note: The $private_ipv4 and $public_ipv4 substitution variables referenced in other documents are not supported when installing via the coreos-install script.


### cloud-config.yamlの再実行

また、CoreOSをディスクにインストールするときに使ったcloud-config.yamlは、`/var/lib/coreos-install/user_data`にインストールされます。この設定はboot毎に再読み込みされるそうです。

> The installation script will process your cloud-config.yaml file specified with the -c flag and place it onto disk. It will be installed to /var/lib/coreos-install/user_data and evaluated on every boot. Cloud-config is not the only supported format for this file — running a script is also available.


ディスクにインストールした後でも、discoveryのTOKENを変更したい場合は、
`/var/lib/coreos-install/user_data`を編集してrebootすれば反映してくれそうなのですが。。。

### rebootで失敗

`/var/lib/coreos-install/user_data`を編集して`COREOS_PRIVATE_IPV4`を記入後にrebootします。

残念ながら、reboot後にUnitの起動が失敗してしまいました。
このUnitファイルはみつからないので、まだ意味がわかりません。

``` bash
CoreOS (beta)
Failed Units: 1
  user-cloudinit@var-lib-coreos\x2dinstall-user_data.service
```

`/run/systemd/system/etcd.service.d/20-cloudinit.conf`がなくなっています。

``` bash
$ cat /run/systemd/system/etcd.service.d/20-cloudinit.conf
cat: /run/systemd/system/etcd.service.d/20-cloudinit.conf: No such file or directory
```

`/etc/environment`ファイルも生成されていません。
その代わり、`/cloudinit-tempxxxx`のファイルが作成されるようになりました。でもempty。

``` bash /cloudinit-temp850627980
COREOS_PUBLIC_IPV4=
COREOS_PRIVATE_IPV4=
```


### write_filesの書式が違っていた

[Using Cloud-Config](http://coreos.com/docs/cluster-management/setup/cloudinit-cloud-config/)のページで書式を確認すると、`write_files`の書き方を間違えていました。正しくは`-name:`でなく`-path:`です。

``` yaml
#cloud-config
write_files:
  - path: /etc/fleet/fleet.conf
    permissions: 0644
    content: |
      verbosity=1
      metadata="region=us-west,type=ssd"
```

`/var/lib/coreos-install/user_data`を編集してrebootします。

``` yaml /var/lib/coreos-install/user_data
write_files:
  - path: /etc/environment
    content: |
        COREOS_PUBLIC_IPV4=$public_ipv4
        COREOS_PRIVATE_IPV4=$private_ipv4
```


reboot後確認すると、`/etc/environment`ファイルはできましたが、まだ値が入りません。

``` bash /etc/environment
COREOS_PUBLIC_IPV4=
COREOS_PRIVATE_IPV4=
```


### 手動でプライベートIPアドレスを指定

* `Update 2014-07-23`

ifconfigでIPアドレスを確認します。

``` bash
$ ifconfig ens32 | awk '/inet /{print $2}'
10.1.0.174
```

確認したIPアドレスを記述します。

``` bash /etc/environment
COREOS_PUBLIC_IPV4=
COREOS_PRIVATE_IPV4=10.1.0.174
```

rebootすると、fleetctlが使えるようになりました。

``` bash
fleetctl list-machines
MACHINE         IP              METADATA
0ba88b13...     10.1.0.174      -
```

`discovery.etcd.io`にもノードが追加されるようになりました。

``` bash
$ curl https://discovery.etcd.io/ed724cf0abd2d02abf985f57b53d6cf4
{"action":"get","node":{"key":"/_etcd/registry/ed724cf0abd2d02abf985f57b53d6cf4","dir":true,"nodes":[{"key":"/_etcd/registry/ed724cf0abd2d02abf985f57b53d6cf4/0ba88b13eef245cc89d1dd880fd8f7fa","value":"http://10.1.0.174:7001","expiration":"2014-07-30T04:43:25.881316985Z","ttl":604662,"modifiedIndex":57181290,"createdIndex":57181290}],"modifiedIndex":56074323,"createdIndex":56074323}}
```

$public_ipv4 と $private_ipv4は、[coreos-cloudinit](https://github.com/coreos/coreos-cloudinit)が直接EC2のmetadataやOpenStackのconfig-driveから取得するように変わったようです。

IDCFクラウドではuser_dataが取得できないので、$public_ipv4 と $private_ipv4が空になっているようです。

> This was intended but we should probably revise it if folks are depending on the old contents of /etc/environment. For EC2/OpenStack instances we moved the detection of $public_ipv4 and $private_ipv4 directly into coreos-cloudinit so that it would work gracefully with both EC2-style metadata services and config drive. The old /usr/share/oem/bin/coreos-setup-environment shipped with those images hung if no metadata service was available, breaking config drive based OpenStack systems.

### 教訓

ドキュメントは普通に全部読むこと。特にCoreOSのドキュメントは頻繁に更新されます。
issuesを読むこと。きっとだれか同じ状況で困ってます。
CoreOSの変更はアグレッシブ。ばさっと大きな変更をしてきます。

