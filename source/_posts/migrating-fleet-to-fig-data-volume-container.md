title: "CoreOSのfleetからfigに移行する - Part2: Data Volume Container"
date: 2014-11-26 22:05:54
tags:
 - fig
 - Ubuntu
 - MoinMoin
 - DataVolumeContainer
description: CoreOSで動かしているMoinMoinはData Volume ContainerにBitTorrent Syncを使っています。Figを使う場合もvolumes_fromのキーを使い定義ができます。
---

CoreOSで動かしている[MoinMoin](https://registry.hub.docker.com/u/masato/moinmoin/)はData Volume Containerに[BitTorrent Sync](https://registry.hub.docker.com/u/masato/btsync/)を使っています。Figを使う場合も[volumes_from](http://www.fig.sh/yml.html)のキーを使い定義ができます。

<!-- more -->


### fleet units

MoinMoinのfleet unitファイルです。これをfig.ymlに変換します。環境変数の`VIRTUAL_HOST`はFQDN、SSLは有効にしている例です。引数はMoinMoinのユーザー名を指定しています。

``` bash ~/docker_apps/moinmoin-system/moinmoin@.service
[Unit]
Description=MoinMoin Service
Wants=%p-data@%i.service
After=%p-data@%i.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill %p%i
ExecStartPre=-/usr/bin/docker rm -v %p%i
ExecStartPre=/usr/bin/docker pull masato/moinmoin
ExecStart=/usr/bin/docker run --name %p%i \
  --volumes-from %p-data%i \
  -e VIRTUAL_HOST=www.example.com \
  -e SSL=true \
  masato/moinmoin ScottTiger
ExecStop=/usr/bin/docker kill %p%i

[Install]
WantedBy=multi-user.target

[X-Fleet]
Conflicts=%p@*.service
MachineMetadata=role=moin
```

同様に、Data Volume Containerのunitファイルもfig.ymlに変換しますが、BitTorrent Syncは使わないようにします。

``` bash ~/docker_apps/moinmoin-system/moinmoin-data@.service
[Unit]
Description=MoinMoin Data Service
Requires=docker.service
After=docker.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill %p%i
ExecStartPre=-/usr/bin/docker rm -v %p%i
ExecStartPre=-/usr/bin/docker pull masato/btsync
ExecStart=/usr/bin/docker run --name %p%i masato/btsync xxx
ExecStop=/usr/bin/docker kill %p%i

[Install]
WantedBy=multi-user.target

[X-Fleet]
Conflicts=%p@*.service
MachineMetadata=role=moin
```

### fig.yml

上記のunitファイルをfig.ymlに書き直します。だいぶすっきりしたYMLファイルになりました。テスト起動なのでSSLは使いません。

``` yml fig.yml
moinmoin:
  image: masato/moinmoin
  command: ScottTiger
  volumes_from:
    - data
  environment:
    VIRTUAL_HOST: www.example.com
data:
  image: ubuntu:14.04
  command: /bin/bash 
  volumes:
    - /data
```

figを使いコンテナを起動します。fleetを使う場合は個別にunitを起動しないといけないのですが`fig up`だけで済むのは便利です。

``` bash
$ fig up -d
Creating root_data_1...
Creating root_moinmoin_1...
$ docker ps -a
CONTAINER ID        IMAGE                    COMMAND              CREATED             STATUS                      PORTS               NAMES
d99cedd6c882        masato/moinmoin:latest   "/run-moinmoin.sh"   19 seconds ago      Up 18 seconds               80/tcp              root_moinmoin_1
4df621c0854d        ubuntu:14.04             "/bin/bash"          19 seconds ago      Exited (0) 18 seconds ago                       root_data_1
```

### MoinMoinデータのリストア

バックアップしている最新のpagesをIDCFオブジェクトストレージからダウンロードします。

``` bash
$ s3cmd get s3://xxx/backups/2014-11-26-17-16-09_pages.tar.gz
$ LATEST_FILE=2014-11-26-17-16-09_pages.tar.gz
```
Data Volume Containerをマウントして、tar.gzを解凍します。

``` bash
$ docker run --rm --volumes-from root_data_1 -v $(pwd):/backup ubuntu /bin/bash -c "rm -fr /data/moin/pages && mkdir -p /data/moin/pages && tar zxf /backup/$LATEST_FILE --strip-components=1 -C /data/moin/pages/ && chown -R www-data:www-data /data/moin/pages/ && chmod -R g+w /data/moin/pages/ && chmod 2775 /data/moin/pages/"
```

userファイルをData Volume Containerにコピーします。

``` bash
$ docker run --rm --volumes-from root_data_1 -v $(pwd):/backup ubuntu /bin/bash -c 'mkdir -p /data/moin/user && cp /backup/user /data/moin/user/xxxxxxxxxx.xx.xxxxx && chown -R www-data:www-data /data/moin/user/ && chmod -R g+w /data/moin/user/ && chmod 2775 /data/moin/user/'
```

docker execコマンドを使い、起動しているコマンドにbashを起動します。

``` bash
$ docker exec -it root_moinmoin_1 /bin/bash
```

elinksからMoinMoinが起動していることを確認します。

``` bash
$ IP_ADDRESS=$(docker inspect --format="{{ .NetworkSettings.IPAddress }}" root_moinmoin_1)
$ elinks $IP_ADDRESS
```

### コンテナの削除

とりあえず動いているようなので、今日の作業の終了としてコンテナは削除します。まず起動しているコンテナをstopします。

``` bash
$ fig stop
Stopping root_moinmoin_1...
```

コンテナを削除します。アタッチしたボリュームも削除します。

``` bash
$ fig rm -v
Going to remove root_moinmoin_1, root_data_1
Are you sure? [yN] y
Removing root_data_1...
Removing root_moinmoin_1...
```