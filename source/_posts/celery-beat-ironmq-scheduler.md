title: "CeleryとIronMQで非同期分散処理 - Part2: celery beatでスケジュール実行"
date: 2014-08-25 01:52:47
tags:
 - Celery
 - IronMQ
 - DockerDevEnv
 - Python
 - 非同期分散
 - Docker監視環境
description: crontabで実行するような繰り返しタスクは、celery beatスケジューラーを使います。Resqueでもcrontabでなくresque-schedulerを使う方が好みです。crontabと比べたスケジュールライブラリの一番のメリットは、プログラムコードと一緒にソースコード管理できるところです。私の場合、別環境にデプロイするときにcrontabを追加するのをよく忘れてしまいます。また、crontab設定のためサーバーにログインする必要がないのも便利です。resque-schedulerと同様に、celery beatは別プロセスとして定期的に指定したタイミングでタスクの予定を入れる働きをします。
---

crontabで実行するような繰り返しタスクは、[celery beat](http://celery.readthedocs.org/en/latest/userguide/periodic-tasks.html)スケジューラーを使います。Resqueでもcrontabでなく[resque-scheduler](https://github.com/resque/resque-scheduler)を使う方が好みです。

crontabと比べたスケジュールライブラリの一番のメリットは、プログラムコードと一緒にソースコード管理できるところです。
私の場合、別環境にデプロイするときにcrontabを追加するのをよく忘れてしまいます。また、crontab設定のためサーバーにログインする必要がないのも便利です。

`resque-scheduler`も同様に、`celery beat`は別プロセスとして定期的に指定したタイミングでタスクの予定を入れる働きをします。

<!-- more -->

### プロジェクトのディレクトリ

テストプロジェクトは以下の構成でつくりました。

``` bash
$ tree ~/python_apps/spike_cron/
/home/docker/python_apps/spike_cron/
|-- __init__.py
|-- celeryconfig.py
|-- mycelery.py
`-- tasks.py
```

前回の[簡単なワーカー](/2014/08/24/celery-ironmq-distributed-asynchronous-task-queue/)サンプルと、パッケージ名以外はほとんど同じです。

基本的には`CELERY_TIMEZONE`へ適切なタイムゾーンを指定して、`CELERYBEAT_SCHEDULE`を書くだけでタスクのスケジュール実行ができます。とても簡単。

``` python ~/python_apps/spike_tasks/celeryconfig.py
CELERY_TIMEZONE = "Asia/Tokyo"

from datetime import timedelta

CELERYBEAT_SCHEDULE = {
    'add-every-30-seconds': {
        'task': 'spike_cron.tasks.add',
        'schedule': timedelta(seconds=30),
        'args': (16, 16)
    },
}
```


### メインプログラム

Celeryのメインプログラムです。パッケージ名以外は前回と同じです。

``` python ~/python_apps/spike_cron/mycelery.py
# -*- coding: utf-8 -*-
from celery import Celery
import iron_celery

app = Celery("tasks",include=['spike_cron.tasks'])
app.config_from_object('spike_cron.celeryconfig')
```

### 設定ファイル

この前の設定ファイルに追記してスケジュールを定義します。

例では30秒間隔で`spike_cron.tasks.add`関数を実行させるため、実行間隔の計算にtimedeltaをimportしてていますが、crontabモジュールを使いcron書式でも書けます。

Pythonスクリプトで動的に可読性の高い設定ファイルを書けるため、JSONやYAMLなどを使わなくても済みます。

``` python ~/python_apps/spike_cron/celeryconfig.py
# -*- coding: utf-8 -*-
BROKER_URL ='ironmq://**:**@'
CELERY_RESULT_BACKEND = 'ironcache://**:**@'
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_ACCEPT_CONTENT = ['json']

CELERY_TIMEZONE = "Asia/Tokyo"

from datetime import timedelta

CELERYBEAT_SCHEDULE = {
    'add-every-30-seconds': {
        'task': 'spike_cron.tasks.add',
        'schedule': timedelta(seconds=30),
        'args': (16, 16)
    },
}
```

### タスクファイル

タスクファイルも前回とパッケージ名以外は同じです。今回のテストでは足し算(add)だけ使います。

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

### cerely beatの起動

`celery beat`を実行します。

``` bash
$ cd ~/python_apps
$ celery -A spike_cron.mycelery beat -l info
celery beat v3.1.13 (Cipater) is starting.
__    -    ... __   -        _
Configuration ->
    . broker -> ironmq://**:**@localhost//
    . loader -> celery.loaders.app.AppLoader
    . scheduler -> celery.beat.PersistentScheduler
    . db -> celerybeat-schedule
    . logfile -> [stderr]@%INFO
    . maxinterval -> now (0s)
[2014-08-22 13:42:41,192: INFO/MainProcess] beat: Starting...
[2014-08-22 13:42:41,231: INFO/MainProcess] Scheduler: Sending due task add-every-30-seconds (spike_cron.tasks.add)
[2014-08-22 13:42:41,239: INFO/MainProcess] Starting new HTTPS connection (1): mq-aws-us-east-1.iron.io
```

### ワーカーの実行

ワーカーの標準出力から、30秒間隔で2回タスクが実行されたことがわかります。

``` bash
$ cd ~/python_apps
$ celery -A spike_cron.mycelery worker -l info
...
[2014-08-22 13:42:45,602: INFO/MainProcess] Received task: spike_cron.tasks.add[e726dc03-af51-4c61-acc5-f9d9690722d7]
[2014-08-22 13:42:45,612: INFO/Worker-1] Starting new HTTPS connection (1): cache-aws-us-east-1.iron.io
[2014-08-22 13:42:45,787: INFO/MainProcess] Starting new HTTPS connection (2): mq-aws-us-east-1.iron.io
[2014-08-22 13:42:46,459: INFO/MainProcess] Task spike_cron.tasks.add[e726dc03-af51-4c61-acc5-f9d9690722d7] succeeded in 0.855676598003s: 32
[2014-08-22 13:43:13,005: INFO/MainProcess] Received task: spike_cron.tasks.add[45ba8c31-3847-47c6-933a-92f79671177b]
[2014-08-22 13:43:13,015: INFO/Worker-2] Starting new HTTPS connection (1): cache-aws-us-east-1.iron.io
[2014-08-22 13:43:13,911: INFO/MainProcess] Task spike_cron.tasks.add[45ba8c31-3847-47c6-933a-92f79671177b] succeeded in 0.904450221002s: 32
```

### まとめ

次は`Fabric API`を使い、リモートサーバーにアクセスするtaskを書いてみます。
Celeryは最初ちょっと敷居が高いですが、慣れてくるととても少ないコード量でやりたいことが実装できます。
もっと前から使えばよかったと思います。

