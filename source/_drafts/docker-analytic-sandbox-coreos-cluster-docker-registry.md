title: "CoreOSクラスタにデータ分析環境をデプロイする - Part1: docker-registryの準備"
date: 2014-08-06 23:37:39
tags:
 - CoreOS
 - fleetctl
 - docker-registry
 - AnalyticSandbox
 - RStudioServer
description:
---

### docker-registryのautostart

最初にdocker-registryコンテナを起動します。

``` bash
$ docker run --name registry -d -p 5000:5000 -v /home/masato/registry_conf:/registry_conf -e DOCKER_REGISTRY_CONFIG=/registry_conf/config.yml -e AWS_S3_ACCESS_KEY="xxx" -e AWS_S3_SECRET_KEY="xxx" -e REGISTRY_SECRET="`openssl rand -base64 64 | tr -d '\n'`" registry
```

Upstartの設定ファイルを書きます。

``` bash /etc/init/docker-registry.conf
description "Docker Registry container"
author "Masato Shimizu"
start on filesystem and started docker
stop on runlevel [!2345]
respawn
script
  /usr/bin/docker start -a docker-registry
end script
```
OSをリブートして、docker-registryが起動するか確認します。

``` bash
$ sudo reboot
```

docker-registryは起動しています。

``` bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED              STATUS              PORTS                    NAMES
de6323f305ed        registry:0.7.3      /bin/sh -c 'exec doc   About a minute ago   Up About a minute   0.0.0.0:5000->5000/tcp   registry

```

docker-registryにcurlで接続します。devモードとバージョンが表示されます。

``` bash
$ curl localhost:5000
"docker-registry server (dev) (v0.7.3)"
```

### RStudio Serverイメージのpush 

`RStudio Server`のイメージをpushします。

``` bash
$ docker tag masato/rstudio-server:latest localhost:5000/rstudio-server
$ docker push localhost:5000/rstudio-server
...
Pushing tag for rev [f9faaccd637d] on {http://localhost:5000/v1/repositories/rstudio-server/tags/latest}
```

### RStudio ServerのUnitファイル作成

``` conf ~/fleetctl_apps/rstudio-server.service
[Unit]
Description=rstudio-server
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/bin/docker run -p 8787:8787 10.1.2.164:5000/rstudio-server /sbin/my_init

[Install]
WantedBy=multi-user.target
```

### fleetctlを使いUnitを起動する

`FLEETCTL_TUNNEL`の環境変数を設定してコマンドを確認します。

``` bash
$ export FLEETCTL_TUNNEL=10.1.0.174
$ eval `ssh-agent`
$ ssh-add ~/.ssh/google_compute_engine
$ fleetctl list-machines
MACHINE         IP              METADATA
0ba88b13...     10.1.0.174      -
3b9bf346...     10.1.1.90       -
5d2a8312...     10.1.0.117      -
```

`fleetctl submit`コマンドを使い、Unitファイルを登録します。

```
$ fleetctl submit rstudio-server.service
$ fleetctl list-units
UNIT                    DSTATE          TMACHINE        STATE           MACHINE ACTIVE
rstudio-server.service  inactive        -               inactive        -       -
```

`fleetctl start`コマンドを使い、Unitを起動します。

``` bash
$ fleetctl start rstudio-server.service
Job rstudio-server.service launched on 3b9bf346.../10.1.1.90
$ fleetctl list-units
UNIT                    DSTATE          TMACHINE                STATE           MACHINE            ACTIVE
rstudio-server.service  launched        3b9bf346.../10.1.1.90   launched        3b9bf346.../10.1.1.90       active
```

### Unitの確認

statusを表示します。

```
$ fleetctl status rstudio-server.service
● rstudio-server.service - rstudio-server
   Loaded: loaded (/run/fleet/units/rstudio-server.service; linked-runtime)
   Active: active (running) since Sun 2014-08-03 07:56:36 UTC; 1min 57s ago
 Main PID: 20888 (docker)
   CGroup: /system.slice/rstudio-server.service
           └─20888 /usr/bin/docker run -p 8787:8787 10.1.2.164:5000/rstudio-server /sbin/my_init

Aug 06 07:56:36 coreos-beta-v3-2 systemd[1]: Started rstudio-server.
Aug 06 07:56:36 coreos-beta-v3-2 docker[20888]: Unable to find image '10.1.2.164:5000/rstudio-server' locally
Aug 06 07:56:38 coreos-beta-v3-2 docker[20888]: Pulling repository 10.1.2.164:5000/rstudio-server
```

journalの確認をします。

```
$ fleetctl journal rstudio-server.service
-- Logs begin at Wed 2014-07-23 09:10:02 UTC, end at Sun 2014-08-03 07:57:38 UTC. --
Aug 06 07:34:13 coreos-beta-v3-2 systemd[1]: Starting rstudio-server...
Aug 06 07:34:13 coreos-beta-v3-2 systemd[1]: Started rstudio-server.
Aug 06 07:34:13 coreos-beta-v3-2 docker[18169]: Unable to find image '10.1.2.164:5000/rstudio-server' locally
Aug 06 07:34:15 coreos-beta-v3-2 docker[18169]: Pulling repository 10.1.2.164:5000/rstudio-server
Aug 06 07:35:58 coreos-beta-v3-2 docker[18169]: 2014/08/03 07:35:58 Could not find repository on any of the indexed registries.
Aug 06 07:35:58 coreos-beta-v3-2 systemd[1]: rstudio-server.service: main process exited, code=exited, status=1/FAILURE
Aug 06 07:35:58 coreos-beta-v3-2 systemd[1]: Unit rstudio-server.service entered failed state.
Aug 06 07:38:26 coreos-beta-v3-2 systemd[1]: Stopped rstudio-server.
Aug 06 07:56:36 coreos-beta-v3-2 systemd[1]: Starting rstudio-server...
Aug 06 07:56:36 coreos-beta-v3-2 systemd[1]: Started rstudio-server.
```

ブラウザで確認します。

{% img center /2014/08/04/docker-analytic-sandbox-coreos-cluster-docker-registy/rstudio-server.png %}


一度Unitを停止します。

```
$ fleetctl destroy rstudio-server.service
Destroyed Job rstudio-server.service
```  
