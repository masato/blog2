title: "CeleryとIronMQで非同期分散処理 - Part1: 簡単なワーカー"
date: 2014-08-24 09:35:03
tags:
 - Celery
 - IronMQ
 - DockerDevEnv
 - Python
 - 非同期分散
 - Docker監視環境
description: いつも非同期分散処理はRailsとResqueワーカーで書いていましたが、一部のワーカーはPythonのPyResを使っていました。今度は新しいPythonプロジェクトなので、最初からResqueではなくCeleryを使ってみようと思います。ブローカーにはAmazonSQSのようなIronMQを使います。IronMQにはLite (Free)プランがあるので気軽に開発を始めることができます。ローカルでRedisやRabbitMQの面倒はもうみたくないので、なるべくクラウドサービスを組み合わせたアーキテクチャを心掛けたいです。
---

いつも非同期分散処理はRailsとResqueワーカーで書いていましたが、一部のワーカーはPythonのPyResを使っていました。今度は新しいPythonプロジェクトなので、最初からResqueではなくCeleryを使ってみようと思います。

ブローカーにはAmazonSQSのようなIronMQを使います。IronMQには[Lite (Free)](http://www.iron.io/pricing)プランがあるので気軽に開発を始めることができます。ローカルでRedisやRabbitMQの面倒はもうみたくないので、なるべくクラウドサービスを組み合わせたアーキテクチャを心掛けたいです。

<!-- more -->

### インストール

[Docker上に構築した](/2014/08/22/linuxmint17-dockerdevenv/)Python開発環境を使います。
通常のPythonの開発でvirtualenvは必須ですが、Docker開発環境なので気にせずシステムワイドにpipを使います。

`iron_celery`と一緒にceleryや日付処理のライブラリ等も一緒にインストールされます。

``` bash
$ sudo pip install iron_celery
...
Successfully installed iron-celery iron-mq iron-cache celery iron-core billiard kombu pytz python-dateutil amqp anyjson
Cleaning up...
```

### プロジェクトのディレクトリ

テスト用のプロジェクトは以下のような構成になります。今回は実行タスクは別ファイルに定義しています。

``` bash
$ tree ~/python_apps/spike_tasks/
/home/docker/python_apps/spike_tasks/
|-- __init__.py
|-- celeryconfig.py
|-- mycelery.py
`-- tasks.py
```

### メインプログラム

Celeryのメインプログラムです。第一引数の"tasks"は任意のアプリケーション名です。
第二引数のincludeにタスクを定義しているパッケージを指定します。

``` python ~/python_apps/spike_tasks/mycelery.py
# -*- coding: utf-8 -*-
from celery import Celery
import iron_celery

app = Celery("tasks",include=['spike_tasks.tasks'])
app.config_from_object('spike_tasks.celeryconfig')
```

### 設定ファイル

プログラムから`config_from_object`でロードするceleryconfig.pyを作成します。

BROKERとRESULT_BACKENDはそれぞれIronMQとIronCacheを指定します。
IronMQの管理画面からCredentialボタンを押しPROJECT_IDとTOKENを取得しておきます。URLは`@`で終わります。

> ironmq://project_id:token@

また、メッセージ書式はJSONのみ許可します。

``` python ~/python_apps/spike_tasks/celeryconfig.py
# -*- coding: utf-8 -*-
BROKER_URL ='ironmq://**:**@'
CELERY_RESULT_BACKEND = 'ironcache://**:**@'
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_ACCEPT_CONTENT = ['json']
```

### タスクファイル

[Getting Started](http://celery.readthedocs.org/en/latest/getting-started/next-steps.html)を参考にして、タスクを3つ定義します。今回のテストではかけ算(mul)だけ使います。

``` python ~/python_apps/spike_tasks/tasks.py
# -*- coding: utf-8 -*-
from spike_tasks.mycelery import app

@app.task
def add(x, y):
    return x + y

@app.task
def mul(x, y):
    return x * y

@app.task
def xsum(numbers):
    return sum(numbers)
```

### ワーカーの起動

メインプログラムを指定してceleryコマンドから実行します。
`AWS US-East`で動いているIronMQに接続してqueueを待機します。

``` bash
$ cd ~/python_apps
$ celery -A spike_tasks.mycelery worker -l info
 -------------- celery@a05a1d9d42ee v3.1.13 (Cipater)
---- **** -----
--- * ***  * -- Linux-3.13.0-24-generic-x86_64-with-Ubuntu-14.04-trusty
-- * - **** ---
- ** ---------- [config]
- ** ---------- .> app:         tasks:0x7f01aa871710
- ** ---------- .> transport:   ironmq://**:**@localhost//
- ** ---------- .> results:     ironcache://**:**@
- *** --- * --- .> concurrency: 4 (prefork)
-- ******* ----
--- ***** ----- [queues]
 -------------- .> celery           exchange=celery(direct) key=celery

[tasks]
  . spike_tasks.tasks.add
  . spike_tasks.tasks.mul
  . spike_tasks.tasks.xsum

[2014-08-22 12:55:32,369: INFO/MainProcess] Connected to ironmq://**:**@localhost//
[2014-08-22 12:55:32,384: INFO/MainProcess] Starting new HTTPS connection (1): mq-aws-us-east-1.iron.io
[2014-08-22 12:55:33,232: INFO/MainProcess] Starting new HTTPS connection (1): mq-aws-us-east-1.iron.io
[2014-08-22 12:55:34,055: WARNING/MainProcess] celery@a05a1d9d42ee ready.
```

### タスクの実行

別のシェルからPythonインタプリタを起動して、タスクを実行します。

``` bash
$ cd ~/python_apps
$ python
Python 2.7.6 (default, Mar 22 2014, 22:59:56)
[GCC 4.8.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> from spike_tasks.tasks import mul
>>> mul.delay(2,3)
<AsyncResult: 17839644-bacb-4b0e-ad32-241151831fd6>
>>>
```

### ワーカーの実行確認

workerの標準出力に非同期処理の結果が出力されます。`2 * 3 = 6`です。

``` bash
...
[2014-08-22 12:57:20,066: INFO/MainProcess] Received task: spike_tasks.tasks.mul[17839644-bacb-4b0e-ad32-241151831fd6]
[2014-08-22 12:57:20,076: INFO/Worker-1] Starting new HTTPS connection (1): cache-aws-us-east-1.iron.io
[2014-08-22 12:57:20,954: INFO/MainProcess] Task spike_tasks.tasks.mul[17839644-bacb-4b0e-ad32-241151831fd6] succeeded in 0.887078803004s: 6
```

### まとめ

プログラム自体はとても簡単ですが、サンプルがうまく動かず少し苦労しました。
次回はCeleryのスケジュール機能を使い、crontabのようにワーカーを実行してみます。