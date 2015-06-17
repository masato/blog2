title: "Project Atomic First Steps - Part3: geardでDockerコンテナ管理"
date: 2014-09-16 01:28:44
tags:
 - Docker管理
 - geard
 - ProjectAtomic
 - DataVolumeContainer
description: デプロイしたAtomic Hostを使い、Dockerコンテナ管理ツールのgeardの動作確認していきます。Dockerを操作するsystemdのunitファイルを生成してくれるところはCoreOSのfleetと同じです。JSONファイルから複数のコンテナをLinkさせてデプロイできるところはfigやPanamax、Kubernetesに似ています。OpenShift Origin 3ではOrchestrationをKubernetesに任せています。geardの概念的なのところは新しいコマンドやKubeletに統合されていくようですが、geard自体はなくなりそうな感じです。
---

デプロイした[Atomic Host](/2014/09/15/projectatomic-first-steps-on-idcf-ova-deploy/)を使い、Dockerコンテナ管理ツールの[geard](http://www.projectatomic.io/docs/geard/)の動作確認していきます。

Dockerを操作するsystemdのunitファイルを生成してくれるところはCoreOSのfleetと同じです。JSONファイルから複数のコンテナをLinkさせてデプロイできるところはfigやPanamax、Kubernetesに似ています。

`OpenShift Origin 3`ではOrchestrationをKubernetesに任せています。geardの概念的なのところは新しいコマンドやKubeletに統合されていくようですが、geard自体はなくなりそうな感じです。


<!-- more -->


### sudo gear daemon

APIリクエストを受け付ける`geard agent`をデーモンとしてroot権限で起動します。
以降のlocalhostへのCLIコマンドは、sudoなしで実行可能になります。

``` bash
$ sudo gear daemon
Listening (HTTP) on :43273 ...
```


### gear install

API経由でエージェントに接続して、systemdのunitファイルを作成します。

``` bash
$ gear install pmorie/sti-html-app localhost/my-sample-service
```

### unitファイル

作成されるunitファイルの場所と書式は以下です。

``` bash
/etc/systemd/system/ctr-<コンテナ名>.service
```

自分で書くと大変ですがgeardがunitファイルを生成してくれます。

``` bash /etc/systemd/system/ctr-my-sample-service.service
[Unit]
Description=Container my-sample-service


[Service]
Type=simple
TimeoutStartSec=5m
Slice=container-small.slice


# Create data container
ExecStartPre=/bin/sh -c '/usr/bin/docker inspect --format="Reusing &#123;&#123;.ID}}" "my-sample-service-data" || exec docker run --name "my-sample-service-data" --volumes-from "my-sample-service-data" --entrypoint true "pmorie/sti-html-app"'
ExecStartPre=-/usr/bin/docker rm "my-sample-service"

ExecStart=/usr/bin/docker run --rm --name "my-sample-service" \
          --volumes-from "my-sample-service-data" \
           \
          -a stdout -a stderr   \
           \
          "pmorie/sti-html-app"
# Set links (requires container have a name)
ExecStartPost=-/usr/bin/gear init --post "my-sample-service" "pmorie/sti-html-app"
ExecReload=-/usr/bin/docker stop "my-sample-service"
ExecReload=-/usr/bin/docker rm "my-sample-service"
ExecStop=-/usr/bin/docker stop "my-sample-service"

[Install]
WantedBy=container.target

# Container information
X-ContainerId=my-sample-service
X-ContainerImage=pmorie/sti-html-app
X-ContainerUserId=
X-ContainerRequestId=x-mIEwrGgAG9M0Agpts39A
X-ContainerType=simple
```


### gear start

コンテナを起動します。

``` bash
$ gear start localhost/my-sample-service
Container my-sample-service starting
$ systemctl status ctr-my-sample-service
ctr-my-sample-service.service - Container my-sample-service
   Loaded: loaded (/var/lib/containers/units/my/ctr-my-sample-service.service; enabled)
   Active: activating (start-pre) since 土 2014-09-13 13:29:43 UTC; 42s ago
  Control: 1014 (docker)
   CGroup: /container.slice/container-small.slice/ctr-my-sample-service.service
           └─control
             └─1014 docker run --name my-sample-service-data --volumes-from my-sample-service-dat...
```

### gear list-units

unitのリストをします。

``` bash
$ gear list-units localhost
ID                SERVER          ACTIVE  SUB     LOAD    TYPE
my-sample-service localhost:43273 active  running loaded
```

`docker ps`でも確認できます。

``` bash
$ docker ps
CONTAINER ID        IMAGE                        COMMAND                CREATED             STATUS              PORTS               NAMES
70ecba9cde14        pmorie/sti-html-app:latest   /bin/sh -c /usr/bin/   9 minutes ago       Up 9 minutes        8080/tcp            my-sample-service
```


### curlで確認

inspectをしてIPアドレスを表示します。

``` bash
$ docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}"  70ecba9cde14
172.17.0.4
```

curlでコンテナが動作を確認します。

``` bash
$ curl 172.17.0.4:8080
<html>
  <head>
    <title>Welcome</title>
  </head>
  <body>
This is a test STI HTML application.
  </body>
</html>
```

### gear stop

`gear stop`でコンテナを停止します。

``` bash
$ gear stop localhost/my-sample-service
 9月 13 13:44:58 i-669-70959-VM systemd[1]: Stopping Container my-sample-service...
 9月 13 13:44:58 i-669-70959-VM docker[1305]: [2014-09-13 13:44:58] INFO  going to shutdown ...
 9月 13 13:44:58 i-669-70959-VM docker[1305]: [2014-09-13 13:44:58] INFO  WEBrick::HTTPServer#start done.
 9月 13 13:44:58 i-669-70959-VM docker[1420]: my-sample-service
 9月 13 13:44:59 i-669-70959-VM systemd[1]: Stopped Container my-sample-service.
Container my-sample-service is stopped
$ docker ps
CONTAINER ID        IMAGE                        COMMAND                CREATED             STATUS                      PORTS               NAMES
8ec9ada499f0        pmorie/sti-html-app:latest   true /bin/sh -c /usr   13 minutes ago      Exited (0) 13 minutes ago                       my-sample-service-data
$ systemctl status ctr-my-sample-service
ctr-my-sample-service.service - Container my-sample-service
   Loaded: loaded (/var/lib/containers/units/my/ctr-my-sample-service.service; enabled)
   Active: inactive (dead)
```

### gear delete

`gear delete`でコンテナを削除します。unitファイルも削除されました。

``` bash
$ gear delete localhost/my-sample-service
Deleted my-sample-service
$ systemctl status ctr-my-sample-service
ctr-my-sample-service.service
   Loaded: not-found (Reason: No such file or directory)
   Active: inactive (dead)
```

### gear deploy

`gear deploy`コマンドを使うと、コンテナとLink情報を記載したJSONファイルから複数のコンテナを起動できます。

[deplo with geard](http://openshift.github.io/geard/deploy_with_geard.html)を参考にして使ってみます。

JSONファイルをダウンロードして中身を表示します。2種類のコンテナを1つずつ起動します。
FigやKubernetesの設定ファイルと似た感じです。

``` bash
$ curl -O \
https://raw.githubusercontent.com/openshift/geard/master/deployment/fixtures/rockmongo_mongo.json
$ cat rockmongo_mongo.json
{
  "containers":[
    {
      "name":"rockmongo",
      "count":1,
      "image":"openshift/centos-rockmongo",
      "publicports":[
        {"internal":80,"external":6060}
      ],
      "links":[
        {"to":"mongodb"}
      ]
    },
    {
      "name":"mongodb",
      "count":1,
      "image":"openshift/centos-mongodb",
      "publicports":[
        {"internal":27017}
      ]
    }
  ]
}
```

`gear deploy`コマンドはroot権限が必要です。

``` bash
$ sudo gear deploy rockmongo_mongo.json
==> Deploying rockmongo_mongo.json
local PortMapping: 80 -> 6060
local Container rockmongo-1 is installed
ports: searching block 41, 4000-4099
ports: Reserved port 4000
local PortMapping: 27017 -> 4000
local Container mongodb-1 is installed
==> Linking rockmongo: 127.0.0.1:27017 -> localhost:4000
local Container rockmongo-1 starting
local Container mongodb-1 starting
==> Deployed as rockmongo_mongo.json.20140913-135111
```

イメージのpullにしばらく時間がかかります。

``` bash
$ docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
openshift/centos-rockmongo   latest              78d2495a2a1d        6 weeks ago         459.7 MB
openshift/centos-mongodb     latest              16d7244a09fd        6 weeks ago         561.6 MB
```

`docker ps`でイメージのダウンロード終了とコンテナの起動を確認します。

``` bash
$ docker ps
CONTAINER ID        IMAGE                               COMMAND                CREATED             STATUS              PORTS                     NAMES
24b27e742ba2        openshift/centos-rockmongo:latest   /usr/sbin/httpd -D F   41 seconds ago      Up 40 seconds       0.0.0.0:6060->80/tcp      rockmongo-1
18b899e6e651        openshift/centos-mongodb:latest     /usr/bin/mongod --co   2 minutes ago       Up About a minute   0.0.0.0:4000->27017/tcp   mongodb-1
```

### ブラウザで確認

ブラウザからRockMongoの管理画面を表示して確認します。
デフォルトの認証情報は以下です。

* user: admin
* password: admin

{% img center /2014/09/16/projectatomic-first-steps-on-idcf-geard/rockmongo.png %}
