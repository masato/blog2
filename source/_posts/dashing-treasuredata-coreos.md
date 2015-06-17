title: 'DashingとTreasure Data - Part5: CoreOSへデプロイ'
date: 2014-06-07 00:10:20
tags:
 - Dashing
 - AnalyticSandbox
 - TreasureData
 - IDCFクラウド
 - IDCFオブジェクトストレージ
 - Docker
 - CoreOS
 - docker-registry
description: Part1,Part2,Part3,Part4に続いてTreasure DataのサンプルアプリをCoreOSにデプロイします。IDCFクラウドに構築したCoreOSのunit fileをsystemdに登録して、CoreOSのリブート後もコンテナが起動するように設定します。少しずつdisposableな環境ができてきました。

---
[Part1](/2014/05/27/dashing-treasuredata-install/),[Part2](/2014/05/30/dashing-treasuredata-td/),[Part3](/2014/05/31/dashing-treasuredata-td-job/)[Part4](/2014/06/06/dashing-treasuredata-docker-registry)に続いて`Treasure Data`のサンプルアプリをCoreOSにデプロイします。
[IDCFクラウドに構築したCoreOS](/2014/06/03/idcf-coreos-install-disk/)の`unit file`をsystemdに登録して、CoreOSのリブート後もコンテナが起動するように設定します。

少しずつdisposableな環境ができてきました。

<!-- more -->

### CoreOSへデプロイ

CoreOSのインスタンスにSSHで接続します。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/private_key
$ ssh -A core@coreos-v3
CoreOS (beta)
```

CoreOSのsytemdに登録する`unit files`のディレクトリは以下です。
``` bash
/etc/systemd/system
```

ここにDashingの`unit file`を作成します。
ExecStartの`docker run`は、`--rm`オプションを追加して、コンテナ停止後に削除されるようにします。
これを指定しないと、CoreOSがrebootしたときにコンテナが重複して起動しなくなります。この辺がdisposableな感じです。

``` bash /etc/systemd/system/dashing.service
[Unit]
Description=Dashing Service
Requires=docker.service
After=docker.service

[Service]
ExecStart=/usr/bin/docker run --rm \
  --name %n \
  -p 80:80 \
  --name dashing \
  -e TD_API_KEY=xxx \
  {private registryのホスト}:5000/dashing /sbin/my_init
ExecStop=/usr/bin/docker stop %n

[Install]
WantedBy=local.target
```

systemctlコマンドを使い、`unit file`をsystemdに登録します。

``` bash
$ sudo systemctl enable /etc/systemd/system/dashing.service
ln -s '/etc/systemd/system/dashing.service' '/etc/systemd/system/local.target.wants/dashing.service'
$ sudo systemctl start dashing.service
```

### まとめ

CoreOSがrebootした後も、コンテナが起動しているところまでdisposableな環境ができました。

次は、[Building Your First App on CoreOS: Start to Finish](http://www.centurylinklabs.com/building-your-first-app-on-coreos/)の参考にしながら、fleetとfigを使ってIDCFクラウド上にCorOSクラスタを構築していみます。