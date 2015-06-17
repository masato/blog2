title: "CeleryとIronMQで非同期分散処理 - Part6: ElephantSQLを使う"
date: 2014-09-02 00:20:11
tags:
 - Celery
 - IronMQ
 - ElephantSQL
 - ClearDB
 - PostgreSQL
 - Docker監視環境
description: PostgreSQL-as-a-ServiceのElaphantSQLをIronMQと一緒に使ってみます。Amazon RDSと同じように、PostgreSQLをクラウドサービスとして利用できます。HerokuやAppHarbor、cloudControlなどのPaaSからボタン一つでインテグレーションできるのが特徴です。ウミガメがかわいいTiny turtleプラン(20MB)は無料で使えます。今回のPoCには十分なサイズです。
---

* `Update 2014-10-09`: [IronMQの非同期処理をNewRelicで監視 - Part7: mackerel-agentをFabricでインストール](/2014/08/27/celery-ironmq-newrelic-in-docker-container/)

`PostgreSQL-as-a-Service`の[ElaphantSQL](http://www.elephantsql.com/)を[IronMQ](http://www.iron.io/mq)と一緒に使ってみます。`Amazon RDS`と同じように、PostgreSQLをクラウドサービスとして利用できます。[Heroku](https://www.heroku.com/)や[AppHarbor](https://appharbor.com/)、[cloudControl](https://www.cloudcontrol.com)などのPaaSからボタン一つでインテグレーションできるのが特徴です。

ウミガメがかわいい`Tiny turtle`プラン(20MB)は無料で使えます。今回のPoCには十分なサイズです。

{% img center /2014/09/02/celery-ironmq-elephantsql/elephantsql.png %}


<!-- more -->


### サインアップ

メールアドレスを入力して[Sign up](https://customer.elephantsql.com/login)ボタンから登録します。
DataCenterは、`Amazon Asia Pacific (Tokyo)`もプルダウンにありますが、`Tiny turtle(Free)`の場合は`Amazon US East (Northern Virginia)`しか選べないようです。

サインアップしてデータベースを作成すると、接続URLを取得できます。

``` bash
postgres://{username}:{passwd}@babar.elephantsql.com:5432/{dbname}
```

### psqlのインストール

``` bash
$ sudo apt-get install postgresql-client-9.3 
```

接続確認します。

``` bash
$ psql postgres://***:***@babar.elephantsql.com:5432/***
psql (9.3.5, server 9.2.8)
SSL connection (cipher: DHE-RSA-AES256-SHA, bits: 256)
Type "help" for help.
***=>
```

### Pythonのpsycopg2モジュール

psycopg2と依存パッケージをインストールします。

``` bash
$ sudo apt-get install libpq-dev
$ sudo pip install psycopg2
...
Successfully installed psycopg2
Cleaning up...
```

環境変数から接続用URLを取得してパースします。

``` python ~/python_apps/spike_nocelery/elephantsql.py
import os
import psycopg2
import urlparse
from pprint import pprint

urlparse.uses_netloc.append("postgres")
url = urlparse.urlparse(os.environ["DATABASE_URL"])

conn = psycopg2.connect(database=url.path[1:],
  user=url.username,
  password=url.password,
  host=url.hostname,
  port=url.port
)

pprint(conn)
conn.close()
```

環境変数に`DATABASE_URL`を指定して実行します。コネクションオブジェクトのダンプが取れました。

``` bash
$ DATABASE_URL=postgres://***:***@babar.elephantsql.com:5432/*** python elephantsql.py
connection object at 0x7f7b4e939c58; dsn: 'dbname=xxx user=xxx password=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx host=babar.elephantsql.com port=5432', closed: 0>
```

### SQLSoupのインストール

[SQLSoup](http://sqlsoup.readthedocs.org/en/latest/)はPython用のORMツールである[SQLAlchemy](http://www.sqlalchemy.org/)から派生したDBアクセスライブラリです。
ORMを使わず`Raw SQL`の実行が必要なときに私はよく使っています。またクラスを定義しなくてもDSLがあるのでSQL操作が簡単になります。

pipでSQLSoupをインストールします。

``` bash
$ sudo pip install sqlsoup
...
Successfully installed sqlsoup SQLAlchemy
Cleaning up...
```

### SQLSoupでテーブル作成

execute()メソッドに`Raw SQL`を渡して実行できます。
また、クラスを定義しなくても以下のようにaccountsテーブルに辞書を渡してinsertもできます。

``` python
db.accounts.insert(account_uuid='1234',host='172.17.0.109')
```

accountsテーブルを作成して、サンプルレコードをinsertしてみます。

``` python ~/python_apps/spike_nocelery/create_accounts_table.py 
# -*- coding: utf-8 -*-
from sqlalchemy import create_engine
import sqlsoup

DATABASE_URL = 'postgres://***:***@babar.elephantsql.com:5432/***'

create_sql = '''
CREATE TABLE IF NOT EXISTS accounts (
  id SERIAL PRIMARY KEY,
  account_uuid TEXT,
  host TEXT
)
'''

db = sqlsoup.SQLSoup(DATABASE_URL)
db.execute(create_sql)
db.commit()

db.accounts.insert(account_uuid='1234',host='172.17.0.109')
db.commit()

accounts = db.accounts.all()
print accounts
```

### Pythonクライアントからqueue

今までcurlでqueueしていましたが、JSON形式のメッセージを渡したいのでIronMQのPythonクライアントを使います。
メッセージは文字列なので、辞書からダンプしてJSON文字列にエンコードします。

``` python ~/python_apps/spike_nocelery/queue.py
# -*- coding: utf-8 -*-
from iron_mq import IronMQ
import json

PROJECT_ID = '***'
TOKEN = '***'

mq = IronMQ(project_id=PROJECT_ID,token=TOKEN)
q = mq.queue('my_queue')
msg = dict(apikey='***',
           account_uuid='1234')
q.post(json.dumps(msg))
```


### IronQueryのpull ワーカー

queueに格納されるメッセージはJSON文字列なので、辞書にデコードしてから使います。
テスト用途なので、エラーが発生してもfinally節でqueueをdeleteします。

SQLSoupから予め作成したElephantSQLのDBに接続します。
IronMQのqueueから取得したメッセージの`account_uuid`を検索条件に、ElephantSQLのテーブルからレコードを取得します。

``` python
# -*- coding: utf-8 -*-
from fabric.context_managers import settings
from fabric.api import (run,put)
from fabric.contrib.files import append
from iron_mq import IronMQ

import sys
import time
import json
import urlparse
import traceback
import sqlsoup

from pprint import pprint

PROJECT_ID = '***'
TOKEN = '***'
HOST = '172.17.0.109'
USER = 'root'
PASSWORD = '***'
DATABASE_URL = 'postgres://***:***@babar.elephantsql.com:5432/***'

mq = IronMQ(project_id=PROJECT_ID,token=TOKEN)
q = mq.queue('my_queue')

def find_host_by(account_uuid):
    db = sqlsoup.SQLSoup(DATABASE_URL)
    account = db.accounts.filter_by(account_uuid=account_uuid).first()
    if account:
        return account.host

def main_loop():
    while True:
        msg = q.get()
        if len(msg['messages']) < 1:
            time.sleep(10)
            continue
        try:
            req = json.loads(msg['messages'][0]['body'])
            apikey = req['apikey']
            account_uuid = req['account_uuid']
            host = find_host_by(account_uuid)
            print(host)
        except Exception, e:
            traceback.print_exc()
        finally:
            q.delete(msg["messages"][0]["id"])

if __name__ == '__main__':
    try:
        main_loop()
    except KeyboardInterrupt:
        print >> sys.stderr, '\nExit.\n'
        sys.exit(0)
```

### まとめ

MQを使った非同期分散処理のPoCでは、IronMQもElephantSQLも無料プランで試しています。

自分でデータベースやMQのミドルウェアを管理しないでも、無料でクラウドサービスが使える便利な世の中になりました。
サーバーの面倒から解放されてとても気持ちが軽くなります。
