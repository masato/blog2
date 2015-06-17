title: "IronMQの非同期処理をNewRelicで監視 - Part7: mackerel-agentをFabricでインストール"
date: 2014-10-09 00:51:31
tags:
 - Python
 - NewRelic
 - Mackerel
 - IronMQ
 - ElephantSQL
 - Fabric
 - Docker監視環境
---

Dockerコンテナに[NewRelicエージェント](/2014/08/27/celery-ironmq-newrelic-in-docker-container/)をインストールしましたが、今度は[Mackerelエージェント](http://help-ja.mackerel.io/entry/howto/install-agent)をインストールしてみます。
[Mackerel](https://mackerel.io/)は[はてな](http://www.hatena.ne.jp/)が提供している監視SaaSです。Freeプランがあるので気軽に試すことができます。

<!-- more -->

### ユースケース

今回は以下のようなユースケースでPoCをします。

* 前提条件
 1. mackerel-agentをインストールするDebianコンテナを1台用意しておきます。
 2. DebianコンテナのIPアドレスを`account_uuid`に紐付けてElephantSQLに登録しておきます。

* ユースケース
 1. ワーカーがIronMQのqueueからJSON形式でメッセージをpullします。
 2. JSON形式でIronMQにenqueueします。
 3. メッセージからデコードした`account_uuid`を検索条件にして、インストール対象のIPアドレスをElephantSQLからクエリします。
 4. ElephantSQLからクエリしたIPアドレスを使い、インストール対象のサーバーにFabricAPIでSSH接続します。
 5. mackerel-agentのdepパッケージのダウンロードとインストールを行います。(常に最新をインストール)


### Pythonのメインプログラム

mail-loopを持ったPythonのメインプログラムです。最初にMackerelにログインして、サービスとロールを登録しておきます。
サービスとロールは`'roles = ["MyService:router"]'`のようにmackerel-agent.confに設定します。

PoCなのでDebianコンテナにはSSHはパスワード接続しますが、本番では鍵認証で接続します。

* USER = 'root'
* PASSWORD = 'mypass'

``` python ~/python_apps/spike_nocelery/newrelic_custom.py
# -*- coding: utf-8 -*-
from fabric.context_managers import settings
from fabric.api import run
from fabric.contrib.files import append
from iron_mq import IronMQ
import sys
import time
import json
import urlparse
import traceback
import sqlsoup
import newrelic.agent
from pprint import pprint
PROJECT_ID = '***'
TOKEN = '***'
USER = 'root'
PASSWORD = 'mypass'
DATABASE_URL = 'postgres://***:***@***.elephantsql.com:5432/***'
mq = IronMQ(project_id=PROJECT_ID,token=TOKEN)
q = mq.queue('my_queue')
db = sqlsoup.SQLSoup(DATABASE_URL)
def find_host_by(account_uuid):
    account = db.accounts.filter_by(account_uuid=account_uuid).first()
    if account:
        return account.host
def install_mackerel(apikey,host):
    with settings(host_string=host,
                  user=USER,password=PASSWORD):
        run('''
        cat /etc/debian_version
        apt-get update
        apt-get -y install curl
        curl -O http://file.mackerel.io/agent/deb/mackerel-agent_latest.all.deb
        dpkg -i --force-confnew mackerel-agent_latest.all.deb
        ''')
        sources = ('apikey = "{0}"'.format(apikey),
                   'roles = ["MyService:router"]')
        append('/etc/mackerel-agent/mackerel-agent.conf',sources)
        run('''
        /etc/init.d/mackerel-agent start
        tail -5 /var/log/mackerel-agent.log
        ''')
def task():
    msg = q.get()
    if len(msg['messages']) < 1:
        time.sleep(10)
        return
    try:
        req = json.loads(msg['messages'][0]['body'])
        print('in task')
        print(req)
        apikey = req['apikey']
        account_uuid = req['account_uuid']
        print('apikey: {0}'.format(apikey))
        print('account_uuid {0}'.format(account_uuid))
        host = find_host_by(account_uuid)
        if host:
            install_mackerel(apikey,host)
    except Exception, e:
        traceback.print_exc()
    finally:
        print("finally,{0}".format(msg["messages"][0]["id"]))
        print(q.delete(msg["messages"][0]["id"]))
def main_loop():
    while True:
        application = newrelic.agent.application()
        name = newrelic.agent.callable_name(task)
        with newrelic.agent.BackgroundTask(application, name):
            task()
if __name__ == '__main__':
    try:
        main_loop()
    except KeyboardInterrupt:
        print >> sys.stderr, '\nExit.\n'
        sys.exit(0)
```

### NewRelicでPythonアプリ監視が動くようになった

[Part4](/2014/08/27/celery-ironmq-newrelic-in-docker-container/)でPythonアプリの監視に失敗していましたが、NewRelicの方に親切にサポートしていただきアプリ監視ができるようになりました。

`while True:`のmain-loopで動かす場合は[BackgroundTaskManager (context manager)](https://docs.newrelic.com/docs/agents/python-agent/customization-extension/python-instrumentation-api#BackgroundTaskManager-context)に書かれているように、`context manager`を使いカスタムの命令を追加する必要があります。

``` python  ~/python_apps/spike_nocelery/newrelic_custom.py
def task():
    msg = q.get()
...
def main_loop():
    while True:
        application = newrelic.agent.application()
        name = newrelic.agent.callable_name(task)
        with newrelic.agent.BackgroundTask(application, name):
            task()
```

### Debianコンテナの用意

DebianのイメージはTutumの[tutum/debian](https://registry.hub.docker.com/u/tutum/debian/)を使います。
PythonのプログラムからFabricAPI経由でパスワードのSSH接続をするため、環境変数でパスワードを設定します。

``` bash
$ docker pull tutum/debian:wheezy
$ docker run --name mackerel -d  -e ROOT_PASS="mypass" tutum/debian:wheezy
```

DebianのIPアドレスを確認します。

``` bash
$ docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" mackerel
172.17.0.5
```

### ElephantSQLのaccount_uuidを更新する

[ElephantSQLを使う](/2014/09/02/celery-ironmq-elephantsql/)の手順でpsqlを使えるようにしておきます。
ElephantSQLにpsqlで接続します。前回起動したときのDebianコンテナとIPアドレスが変更になっているのでUPDATEします。

``` bash
$ psql postgres://***:***@***.elephantsql.com:5432/***
psql (9.3.5, server 9.2.8)
SSL connection (cipher: DHE-RSA-AES256-SHA, bits: 256)
Type "help" for help.
***=> select * from accounts;
 id | account_uuid |    host
----+--------------+------------
  2 | 1234         | 172.17.0.109
(1 row)
***=> update accounts set host='172.17.0.5' where id = 2;
```

### runitのサービス再起動

runitのサービスではnewrelic-adminを経由して、Pythonのプログラムを起動します。
newrelic-adminをインストールします。

``` bash
$ sudo pip install newrelic
```

カレントディレクトリに、`license_key`を指定してnewrelic.iniの設定ファイルを作成します。

``` bash
$ cd ~/python_apps/spike_nocelery
$ newrelic-admin generate-config {license_key} newrelic.ini
```

環境変数をファイルに記述して、`-e`フラグでディレクトリを指定して使います。

``` bash ~/python_apps/spike_nocelery/env/NEW_RELIC_CONFIG_FILE
newrelic.ini
```

NewRelicのApplicationsページに`Python NoCelery`という名前でアプリケーションが表示されます。

``` bash /etc/service/spike/run
#!/bin/bash
set -eo pipefail
exec 2>&1
appdir=/home/docker/python_apps/spike_nocelery
cd $appdir && \
  exec chpst -u docker -e env \
  newrelic-admin run-python newrelic_custom.py
```

ログの設定です。

``` bash /etc/service/spike/log/run
#!/bin/bash
set -eo pipefail
service=$(basename $(dirname $(pwd)))
logdir="/var/log/${service}"
mkdir -p ${logdir}
exec 2>&1
exec /usr/bin/svlogd -tt ${logdir}
```

ruintのサービスを再起動します。

``` bash
$ sudo sv restart spike
ok: run: spike: (pid 11430) 0s
```

### tail -f

runitで出力したログを`tail -f`します。

``` bash
$ sudo tail -f /var/log/spike/current
```

### JSONをenqueueする

`apikey`と`account_uuid`をキーにしたJSON形式のメッセージを、IronMQにenqueueします。

``` python  ~/python_apps/spike_nocelery/queue.py
# -*- coding: utf-8 -*-
from iron_mq import IronMQ
import json

def main():
    PROJECT_ID = 'xxx'
    TOKEN = 'xxx'
    mq = IronMQ(project_id=PROJECT_ID,token=TOKEN)
    q = mq.queue('my_queue')
    msg = dict(apikey='xxx',
           account_uuid='1234')
    ret = json.dumps(msg)
    print(ret)
    q.post(ret)
if __name__ == '__main__':
    main()
```

Pythonスクリプトを実行します。

``` bash
$ python queue.py
{"apikey": "xxx", "account_uuid": "1234"}
```

### ログの確認

10秒間隔でメッセージをpollingしているので、すぐにインストール処理が開始します。

``` bash
$ sudo tail -f /var/log/spike/current
2014-10-08_13:13:14.51419 in task
...
2014-10-08_13:13:18.38520 [172.17.0.5] out: Preparing to replace mackerel-agent 0.12.2-2 (using mackerel-agent_latest.all$deb) ...
2014-10-08_13:13:18.38521 [172.17.0.5] out: Unpacking replacement mackerel-agent ...
2014-10-08_13:13:18.41663 [172.17.0.5] out: Setting up mackerel-agent (0.12.2-2) ...
...
2014-10-08_13:13:18.95541 [172.17.0.5] run:
2014-10-08_13:13:18.95542         /etc/init.d/mackerel-agent start
2014-10-08_13:13:18.95542         tail -5 /var/log/mackerel-agent.log
2014-10-08_13:13:18.95543
2014-10-08_13:13:18.95543 [172.17.0.5] out: Starting mackerel-agent: failed!
2014-10-08_13:13:21.95972 [172.17.0.5] out: 2014/10/08 12:04:36 INFO command Start: apibase = https://mackerel.io, hostName = fbe293fe4694, hostId = 2aYC54vyvGL
2014-10-08_13:13:21.95973 [172.17.0.5] out: 2014/10/08 12:05:04 INFO main Starting mackerel-agent version:0.12.2, rev:67cbc69
2014-10-08_13:13:21.95973 [172.17.0.5] out: 2014/10/08 12:09:34 INFO main Starting mackerel-agent version:0.12.2, rev:67cbc69
2014-10-08_13:13:21.95974 [172.17.0.5] out: 2014/10/08 12:09:34 INFO command Start: apibase = https://mackerel.io, hostName = fbe293fe4694, hostId = 2aYC54vyvGL
2014-10-08_13:13:21.95974 [172.17.0.5] out: 2014/10/08 13:13:18 INFO main Starting mackerel-agent version:0.12.2, rev:67cbc69
2014-10-08_13:13:21.95975 [172.17.0.5] out:
```

### Mackerelのダッシュボード

ダッシュボードにログインすると、エージェントが収集したメトリクスがHostsページにグラフが表示されました。

{% img center /2014/10/09/celery-ironmq-mackerel-in-docker-container-with-newrelic/mackerel.png %}
