title: "CeleryとIronMQで非同期分散処理 - Part4: NewRelic in Docker Container"
date: 2014-08-27 20:31:31
tags:
 - Docker監視環境
 - Celery
 - NewRelic
 - Centurion
 - IronMQ
 - runit
 - DockerDevEnv
 - Python
 - Debian
description: NewRelicがオープンソースにしたDockerコンテナ管理ツールの今度Centurionを試してみようと思います。その前にNewRelicはしばらく使っていませんでした。NewRelicからDockerコンテナもシステム監視ができるようになっているようです。そういえば2008年くらいにRailsで使って以来です。今回はPythonアプリを監視してみます。
---

* `Update 2014-10-09`: [IronMQの非同期処理をNewRelicで監視 - Part7: mackerel-agentをFabricでインストール](/2014/08/27/celery-ironmq-newrelic-in-docker-container/)

NewRelicがオープンソースにしたDockerコンテナ管理ツールのCenturionを今度試してみようと思います。その前にNewRelicはしばらく使っていませんでした。NewRelicからDockerコンテナもシステム監視ができるようになっているようです。そういえば2008年くらいにRailsで使って以来です。今回はPythonアプリを監視してみます。

<!-- more -->

### UbuntuのDockerコンテナ

OSのシステム監視をするため、監視対象のDockerコンテナへnewrelic-sysmondをインストールします。
Ubuntuのリポジトリを追加してインストールします。

``` bash
$ sudo -i
# echo deb http://apt.newrelic.com/debian/ newrelic non-free >> /etc/apt/sources.list.d/newrelic.list
# wget -O- https://download.newrelic.com/548C16BF.gpg | apt-key add -
# apt-get update
# apt-get install newrelic-sysmond
```

設定ファイルを`/etc/newrelic/nrsysmond.cfg`に作成をします。
`license_key`はサインアップ後に取得できます。

``` bash
# nrsysmond-config --set license_key=xxx
```

### runitのinitファイル

phusion/baseimageを使っているのでコンテナのinitはrunitです。最近initをDockerで使うのは役割の分離的にちょっと違う気がしてきましたが、Docker開発環境ではssh-agentを使いたいのでこのままです。

nrsysmond用のinitファイルを作成します。`-f`フラグでフォアグラウンド実行させます。

``` bash /etc/service/nrsysmond/run
#!/bin/bash
set -eo pipefail
exec 2>&1
/usr/sbin/nrsysmond -c /etc/newrelic/nrsysmond.cfg  -l /dev/stdout -f
```

同様にsvlogdのinitファイルです。

``` bash /etc/service/nrsysmond/log/run
#!/bin/bash
set -eo pipefail
service=$(basename $(dirname $(pwd)))
logdir="/var/log/${service}"
mkdir -p ${logdir}
exec 2>&1
exec /usr/bin/svlogd -tt ${logdir}
```

runファイルには忘れず実行権限をつけます。

``` bash
$ sudo chmod +x /etc/service/nrsysmond/run
$ sudo chmod +x /etc/service/nrsysmond/log/run
```

runitをrestartするため、dockerホストからコンテナを`docker restart`します。

```
$ docker restart a05a1d9d42ee
```

### Python Agentのインストール

この前作成した[IronMQとFabricアプリ](/2014/08/26/celery-ironmq-fabric/)をNewRelicの`Python Agent`で監視してみます。

newrelic-adminをインストールします。

``` bash
$ sudo pip install newrelic
...
    changing mode of /usr/local/bin/newrelic-admin to 755
    Installing newrelic-admin script to /usr/local/bin
Successfully installed newrelic
Cleaning up...
```

カレントディレクトリに、`license_key`を指定してnewrelic.iniの設定ファイルを作成します。

``` bash
$ cd ~/python_apps/spike_nocelery
$ newrelic-admin generate-config {license_key} newrelic.ini
```

### Pythonアプリの監視

newrelic-adminコマンド経由で起動するようにrunitのinitファイルを修正します。

``` bash /etc/service/spike/run
#!/bin/bash
set -eo pipefail
exec 2>&1
appdir=/home/docker/python_apps/spike_nocelery
cd $appdir && \
  exec chpst -u docker -e env \
  newrelic-admin run-python tasks.py
```

runitでは環境変数をファイルに記述して、`-e`フラグでディレクトリを指定して使います。

``` bash ~/python_apps/spike_nocelery/env/NEW_RELIC_CONFIG_FILE
newrelic.ini
```

### 監視は失敗

newrelic.iniでログをdebugモードにしているため、詳細なログが出力されます。

``` bash /tmp/newrelic-python-agent.log
2014-08-27 14:07:09,364 (50/MainThread) newrelic DEBUG - Initializing Python agent logging.
2014-08-27 14:07:09,364 (50/MainThread) newrelic DEBUG - Log file "/tmp/newrelic-python-agent.log".
2014-08-27 14:07:09,364 (50/MainThread) newrelic.config DEBUG - agent config log_file = '/tmp/newrelic-python-agent.log'
```

ログも出力され、Pythonアプリも正常に動作していますが、NewRelicコンソールには表示されません。

> This server isn't hosting any apps that report to New Relic

監視対象のPythonモジュールを使っていないためか、バックグラウンドのCelery、WebアプリのDjangoやFlaskのでないと`Python Aegnt`を監視してくれないのかも知れません。

後でこのPythonアプリではデータベース接続もしていくので、監視できるようになるか見ていきます。
