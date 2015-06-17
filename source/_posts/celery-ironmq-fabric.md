title: "CeleryとIronMQで非同期分散処理 - Part3: ワーカーからFabricでSSH接続"
date: 2014-08-26 12:31:31
tags:
 - Celery
 - IronMQ
 - DockerDevEnv
 - Python
 - 非同期分散
 - runit
 - Docker監視環境
description: IronMQを使いワーカーへのタスクのqueueを分散環境から行いたいのが目的です。シリーズのタイトルにCeleryとしていますが今回は使いません。CeleryはPythonのプログラムからdelayメソッドでタスクの非同期実行をしますが、分散環境にPythonの実行環境がなかったり、他システムとの連携を考えると、IronMQにREST APIでメッセージをqueueできると便利です。
---

* `Update 2014-10-09`: [IronMQの非同期処理をNewRelicで監視 - Part7: mackerel-agentをFabricでインストール](/2014/08/27/celery-ironmq-newrelic-in-docker-container/)

IronMQを使いワーカーへのタスクのqueueを分散環境から行いたいのが目的です。シリーズのタイトルにCeleryとしていますが今回は使いません。CeleryはPythonのプログラムからdelayメソッドでタスクの非同期実行をしますが、分散環境にPythonの実行環境がなかったり、他システムとの連携を考えると、IronMQに`REST API`でメッセージをqueueできると便利です。

<!-- more -->

### Celeryのメッセージフォーマット

Celeryの[メッセージフォーマット](http://celery.readthedocs.org/en/latest/internals/protocol.html#message-format)の仕様によるとJSON書式は以下です。

``` json
{"id": "4cc7438e-afd4-4f8f-a2f3-f46567e7ca77",
 "task": "celery.task.PingTask",
 "args": [],
 "kwargs": {},
 "retries": 0,
 "eta": "2009-11-17T12:30:56.527191"}
```

このメッセージをqueueするとタスクを実行できそうなのですが、IronMQをブローカーにしたときの使い方がよくわかりませんでした。
そのうち理解できてきたら、Celeryも試してみます。

今回はRackspaceの`Developer Blog`の[Using IronMQ for Delayed Processing and Increasing Scale](https://developer.rackspace.com/blog/using-ironmq/)というポストを参考に、IronMQとPythonのメインループだけで実装してみようと思います。

IronMQには`REST API`があるので、curlでもメッセージが送れます。キューをPUTする場合は以下のようになります。
OAuthの`TOKEN`と`PROJECT_ID`をIronMQの画面にログインして取得しておきます。

``` bash
$ curl -H "Content-Type: application/json" -H "Authorization: OAuth ***" -d '{"messages":[{"body":"hello world!"}]}' "https://mq-aws-us-east-1.iron.io/1/projects/***/queues/my_queue/messages"
```

このqueueをワーカーからpollingして`pull queue`します。`push queue`のワーカーを作る場合は、IronMQからHTTP/HTTPSでコールバック用のエンドポイントを公開しないといけないので、社内用途では使えません。

テスト用に簡単なPythonプログラムを書きました。`main loop`でpollingしてqueueにメッセージがPUTされたらGETして処理をするようにします。

### プロジェクト

IronMQのライブラリをpipインストールします。

``` bash
$ sudo pip install iron_mq
```

作成したPythonスクリプトは一つです。

``` bash
$ tree ~/python_apps/spike_nocelery/
/home/docker/python_apps/spike_nocelery/
`-- tasks.py
```

### Fabricのインストール

Fabricをpipインストールします。サンプルプログラムはqueueからメッセージを取得できたら、リモートサーバーにSSH接続をして
任意のコマンドを実行するシナリオです。

直接Paramikoを使ってSSH接続をしたり、ファイル転送をしても良いですが、Fabricにはリモートサーバー管理の自動化用の関数が揃っています。PythonのプログラムからAPIを実行できると、サーバー管理がプログラマブルになってとても便利です。

``` bash ~/python_apps/spike_nocelery/tasks.py
$ sudo pip install fabric
...
Successfully installed fabric paramiko pycrypto ecdsa
Cleaning up...
```

### Pythonのメインループ

SSH接続するサーバーは、CentOSなので簡単に`/etc/redhat-release`をcatしてみます。

メインループでqueueにメッセージがあるかpollingして、取得できたら標準出力します。
実際のプロジェクトではSSH接続して実行するコマンドの引数を、分散環境からメッセージで渡します。

処理が終わったら忘れずにqueueからメッセージを削除します。

``` python
# -*- coding: utf-8 -*-
from fabric.context_managers import settings
from fabric.api import (run,put)
from iron_mq import IronMQ
import sys
import time

PROJECT_ID = '***'
TOKEN = '***'
HOST = '10.1.1.73'
USER = 'mshimizu'
KEY_FILENAME = '/home/docker/.ssh/deis'

mq = IronMQ(project_id=PROJECT_ID,
            token=TOKEN)
q = mq.queue('my_queue')

def main_loop():
    while True:
        msg = q.get()
        if len(msg["messages"]) < 1:
            time.sleep(10)
            continue
        print(msg)
        with settings(host_string=HOST,
                      user=USER,
                      key_filename=KEY_FILENAME):
            run('cat /etc/redhat-release')
            q.delete(msg["messages"][0]["id"])

if __name__ == '__main__':
    try:
        main_loop()
    except KeyboardInterrupt:
        print >> sys.stderr, '\nExit.\n'
        sys.exit(0)
```

### 起動テスト

テスト用にフォアグラウンドでプログラムを実行します。

```
$ cd ~/python_apps/spike_nocelery/
$ python tasks.py
```

curlでメッセージをqueueにPUTします。今回はワーカーをキックするだけなので、メッセージの内容に意味はありません。

``` bash
$ curl -H "Content-Type: application/json" -H "Authorization: OAuth ***" -d '{"messages":[{"body":"hello world!"}]}' "https://mq-aws-us-east-1.iron.io/1/projects/***/queues/my_queue/messages"
```

ワーカーの標準出力に、リモートホストにSSH接続してcatを実行した結果が表示されます。

```
{u'messages': [{u'body': u'hello world!', u'reserved_count': 1, u'push_status': {}, u'id': u'6051713210758384919', u'timeout': 60}]}
[10.1.1.73] run: cat /etc/redhat-release
[10.1.1.73] out: CentOS release 6.4 (Final)
[10.1.1.73] out:
```

### runitでデモナイズ

runitの起動スクリプトを用意します。

``` bash /etc/service/spike/run
#!/bin/bash
set -eo pipefail
exec 2>&1
appdir=/home/docker/python_apps/spike_nocelery
cd $appdir && exec chpst -u docker python tasks.py
```

runitに付属しているsvlogdコマンドを使ったログの設定をします。

``` bash /etc/service/spike/log/run
#!/bin/bash
set -eo pipefail
service=$(basename $(dirname $(pwd)))
logdir="/var/log/${service}"
mkdir -p ${logdir}
exec 2>&1
exec /usr/bin/svlogd -tt ${logdir}
```

runitを有効にするため、一度コンテナをrestart します。

```
$ docker restart a05a1d
```

サービスのログをtailします。

```
$ sudo tail -f /var/log/spike/current
```

メッセージをqueueにPUTします。

``` bash
$ curl -H "Content-Type: application/json" -H "Authorization: OAuth ***" -d '{"messages":[{"body":"hello world!"}]}' "https://mq-aws-us-east-1.iron.io/1/projects/***/queues/my_queue/messages"
```

ログにも同様にcatの結果が出力されます。

``` bash /var/log/spike/current
2014-08-26_03:42:23.60780 {u'messages': [{u'body': u'hello world!', u'reserved_count': 1, u'push_status': {}, u'id': u'6051714293089978639', u'timeout': 60}]}
2014-08-26_03:42:23.60784 [10.1.1.73] run: cat /etc/redhat-release
2014-08-26_03:42:23.60785 [10.1.1.73] out: CentOS release 6.4 (Final)
2014-08-26_03:42:23.60786 [10.1.1.73] out:
```