title: "RethinkDB on Docker - Part1: 30 Seconds"
date: 2015-05-25 21:25:56
tags:
 - RethinkDB
 - Docker
 - ngrok
description: ClojureのWebアプリのJSONストレージとしてRethinkDBを使ってみようと思います。まずは何も考えずにさくっとDockerで起動してみます。30秒でできるQuick Startとやってみる気になります。developer friendlyを謳うだけのことがありドキュメントサイトがとても充実しています。
---

ClojureのWebアプリのJSONストレージとしてRethinkDBを使ってみようと思います。まずは何も考えずにさくっとDockerで起動してみます。[30秒でできるQuick Start](http://www.rethinkdb.com/docs/quickstart/)とやってみる気になります。developer friendlyを謳うだけのことがあり[ドキュメントサイト](http://www.rethinkdb.com/docs/)がとても充実しています。

<!-- more -->

## Dockerイメージ

RethinkDBには[オフィシャルイメージ](https://registry.hub.docker.com/_/rethinkdb/)があります。latestのバージョンは`2.0.2`でした。

Docker Hubからイメージをpullしてからカレントディレクトリにデータボリュームをマップしてrunします。

``` bash
$ mkdir ~/rethinkdb_apps
$ cd !$
$ docker pull rethinkdb
$ docker run --name rethinkdb -v "$PWD:/data" -d rethinkdb
```

起動に成功しました。

``` bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                            NAMES
8ab92f373592        rethinkdb:latest    "rethinkdb --bind al   8 seconds ago       Up 7 seconds        8080/tcp, 28015/tcp, 29015/tcp   rethinkdb
```

## ngrok

今回はお試しなので[ngrok](https://ngrok.com/)でトンネルしてクラウドで動かしているDockerコンテナにローカルから接続します。

inspectしてコンテナのIPアドレスを確認します。

``` bash
$ RETHINK_IP=$(docker inspect --format={{ .NetworkSettings.IPAddress }}" rethinkdb)
$ echo $RETHINK_IP
172.17.0.249
```

[ngrok](https://registry.hub.docker.com/u/wizardapps/ngrok/)のイメージをpullして起動します。トンネルしてアクセスできるランダムなURLを生成してくれます。

``` bash
$ docker pull wizardapps/ngrok
$ docker run -it --rm wizardapps/ngrok:latest ngrok $RETHINK_IP:8080
ngrok                                                           (Ctrl+C to quit)

Tunnel Status                 online
Version                       1.7/1.7
Forwarding                    http://3e826126.ngrok.com -> 172.17.0.251:8080
Forwarding                    https://3e826126.ngrok.com -> 172.17.0.251:8080
Web Interface                 0.0.0.0:4040
# Conn                        0
Avg Conn Time                 0.00ms
```

## 30秒でできるQuick Start

[Thirty-second quickstart with RethinkDB](http://www.rethinkdb.com/docs/quickstart/)のページをみながらRethinkDBを触ってみます。

ngrokがトンネルしてくれるURLにブラウザでアクセスします。

https://3e826126.ngrok.com/

シンプルできれいなUIの管理画面です。

![rethinkdb-web-admin.png](/2015/05/25/rethinkdb-on-docker/rethinkdb-web-admin.png)

### テーブルの作成

`Data Exploler`画面に移動します。上のテキストエリアにテーブルを作成するコードを入力します。Runボタンまたは`Shift+Enter`キーを押すと実行されます。


```js
r.db('test').tableCreate('tv_shows')
```

![tableCreate.png](/2015/05/25/rethinkdb-on-docker/tableCreate.png)

`Tree View`にJSON形式で処理結果が表示されました。

```json
{
  "config_changes": [
    {
      "new_val": {
        "db": "test",
        "durability": "hard",
        "id": "25206b5d-3b66-4a48-b99f-16c8b74b418d",
        "name": "tv_shows",
        "primary_key": "id",
        "shards": [
          {
            "primary_replica": "8ab92f373592_i07",
            "replicas": [
              "8ab92f373592_i07"
            ]
          }
        ],
        "write_acks": "majority"
      },
      "old_val": null
    }
  ],
  "tables_created": 1
}
```

### レコードの登録

作成した`tv_shows`テーブルにレコードを2件登録します。一度のinsert関数で複数件の登録ができます。

```js
r.table('tv_shows').insert([{ name: 'Star Trek TNG', episodes: 178 },
                            { name: 'Battlestar Galactica', episodes: 75 }])
```

`Tree View`にinsertの結果が表示されます。

```json
{
  "deleted": 0,
  "errors": 0,
  "generated_keys": [
    "834b3df0-d49b-4159-a202-d23d2c60c1eb",
    "73d5d449-d8cf-4b5a-b016-fff4152d787d"
  ],
  "inserted": 2,
  "replaced": 0,
  "skipped": 0,
  "unchanged": 0
}
```

### レコードのカウントとクエリ

count()でカウントします。

``` js
r.table('tv_shows').count()
```

結果はinsertしたレコード数の2です。

```js
2
```

フィルタに条件を指定してクエリします。

```js
r.table('tv_shows').filter(r.row('episodes').gt(100))
```

`episodes`のフィールドが100より大きい条件なので該当は1件です。

```json
  {
    "episodes": 178,
    "id": "834b3df0-d49b-4159-a202-d23d2c60c1eb",
    "name": "Star Trek TNG"
  }
```

`Raw view`のタブに切り換えると表形式で見やすく表示してくれます。

![query.png](/2015/05/25/rethinkdb-on-docker/query.png)
