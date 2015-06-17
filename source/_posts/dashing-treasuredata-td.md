title: 'DashingとTreasure Data - Part2: Treasure Data Toolbeltのセットアップ'
date: 2014-05-30 00:52:41
tags:
 - Dashing
 - AnalyticSandbox
 - TreasureData
 - DockerEnv
 - rbenv
 - Putty
 - byobu
description: 前回はDashingをインストールして同梱されているダッシュボードを表示するところまで確認しました。Treasure Data Serviceのデータベースに貯めたデータをダッシュボードに表示することがこのシリーズの目標です。次はTreasure Dataへのサインアップと、開発環境へtoolbeltのインストールをします。今回はtdコマンドを使い簡単なクエリ実行までしてみます。
---

[Part1](/2014/05/27/dashing-treasuredata-install/)はDashingをインストールして同梱されているダッシュボードを表示するところまで確認しました。
`Treasure Data Service`のデータベースに貯めたデータをダッシュボードに表示することがこのシリーズの目標です。
次は`Treasure Data`へのサインアップと、開発環境へtoolbeltのインストールをします。今回はtdコマンドを使い簡単なクエリ実行までしてみます。

<!-- more -->


## Treasure Data Serviceの無料サインアップ
`Treasure Data Service`[無料版のサインアップ](http://www.treasuredata.com/jp/products/try-it-now.php)ができるので、すぐに試してみることができます。

## Puttyの確認

複数のPCからPuttyを使っていますが、たまにbyobuのF2キーで新しいウィンドウが開かなくなりました。
あらためて設定を確認します。
* 「接続」 -> 「データ」 -> 「端末タイプを表す文字列」 -> 「 xterm 」
* 「端末」 -> 「キーボード」 -> 「ファンクションキーとキーパッド」 -> 「Xterm R6」

## Docker開発環境の準備

いつものようにssh-agentに鍵を追加します。
``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/private_key
```

コンテナを起動します。Thinサーバーのポートは、Dockerホストへは3031でバインドします。

``` bash
$ ~/docker_apps/phusion/
$ docker run -t -i -p 3031:3030 -v ~/docker_apps/workspaces/dashing:/root/sinatra_apps masato/baseimage:1.0 /sbin/my_init /bin/bash
```

byobuを起動してEmacsで便利に使えるように設定します。

* byobu-config -> エスケープシーケンスの変更 -> ctrl + t
* byobu-config -> ステータス通知の切り替え -> logo(off), ip_address(on)
* byobu-ctrl-a -> (2) Emacs mode
* byobu-select-backend -> tmux

## Treasure Data Toolbelt のインストール
`Treasure Data Service`をコンソールから利用するたのコマンドラインツールをインストールします。
[Treasure Data toolbelt](http://toolbelt.treasuredata.com/)からLinuxのアイコンをクリックします。

[インストールガイド](http://docs.treasuredata.com/articles/installing-the-cli)のページから、Ubuntuのセクションに従います。

インストールコマンドを実行すると、tdコマンドが使えるようになります。

```
$ curl -L http://toolbelt.treasuredata.com/sh/install-ubuntu-precise.sh | sh
$ td help
Additional commands, type "td help COMMAND" for more details:

  help:all                                   # Show usage of all commands
  help <command>                             # Show usage of a command
```

### tdコマンドを使う

最初にtdコマンドが使えるようにアカウントのセットアップをします。
サインアップで入力した、EmailとPasswordを入力します。
``` bash
$ td account -f
Enter your Treasure Data credentials.
Email: ma6ato@gmail.com
Password (typing will be hidden):
Authenticated successfully.
Use 'td db:create <db_name>' to create a database.
```

あとでRubyのプログラムから利用するapikeyを確認します。
```
$ td apikey:show
xxx
$ cat ~/.td/td.conf
[account]
  user = ma6ato@gmail.com
  apikey = xxx
```

### サンプルデータベースの作成

データベースを作成します。アカウントのセットアップをしたときにガイドに表示されました。
ここでは、testdbと名前をつけます。

``` bash
$ td db:create testdb
Database 'testdb' is created.
Use 'td table:create testdb <table_name>' to create a table.
$ td db:list
+--------+-------+
| Name   | Count |
+--------+-------+
| testdb | 0     |
+--------+-------+
1 row in set
```

www_accessという名前で、testdbにテーブルを作成します。
``` bash
$ td table:create testdb www_access
Table 'testdb.www_access' is created.
$ td table:list
+----------+------------+------+-------+--------+-------------+--------------------+--------+
| Database | Table      | Type | Count | Size   | Last import | Last log timestamp | Schema |
+----------+------------+------+-------+--------+-------------+--------------------+--------+
| testdb   | www_access | log  | 0     | 0.0 GB |             |                    |        |
+----------+------------+------+-------+--------+-------------+--------------------+--------+
1 row in set
```

### サンプルテーブルの作成

まだ`Treasure Data Service`へデータを保存していないので、サンプルテーブルを作って動作確認します。
サンプル用の5000レコードのJSONファイルを作成します。

``` bash
$ td sample:apache
Create apache.json with 5000 records whose time is
from 2014-05-29 20:54:12 +0900 to 2014-05-30 14:46:42 +0900.
Use 'td table:import <db> <table> --json apache.json' to import this file.
$ tail -1 apache.json
{"host":"200.129.205.208","user":"-","method":"GET","path":"/category/electronics","code":200,"referer":"-","size":62,"agent":"Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/535.11 (KHTML, like Gecko) Chrome/17.0.963.56 Safari/535.11","time":1401364535}
```

先ほど作成したテーブルにJSONデータをインポートします。
``` bash
$ td table:import testdb www_access --json apache.json
importing apache.json...
  uploading 117354 bytes...
  imported 5000 entries from apache.json.
done.
```

### サンプルクエリ

プロジェクトの作成
``` bash
$ mkdir -p ~/sinatra_apps/sql
$ cd !$
```

`Treasure Data Service`はスキーマレスなデータベースが特徴で、先ほどインポートしたJSONファイルから
事前のスキーマ定義が不要で、自動的にスキーマを作成してデータを保存できます。

``` bash
$ td table:show testdb www_access
Name      : testdb.www_access
Type      : log
Count     : 5000
Schema    : (
    host:string
    path:string
    method:string
    referer:string
    code:long
    agent:string
    user:string
    size:long
)
```

どのようなカラムがあるか確認できたので、
``` sql ~/sinatra_apps/sql/sample.sql
SELECT TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS day
     , COUNT(1) AS cnt
  FROM www_access
 GROUP BY TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')
 ORDER BY day DESC
```

sample.sqlのクエリを実行してみます。無料プランなのでクエリに時間がかかりますが、バッチ実行なら使えそうです。

```
$ td query -w -d testdb -q sample.sql
...
  Time taken: 52.585 seconds
Status     : success
Result     :
+------------+------+
| day        | cnt  |
+------------+------+
| 2014-05-30 | 4350 |
| 2014-05-29 | 650  |
+------------+------+
2 rows in set
```

### まとめ

`Treasure Data Service`が利用できるようになりました。セットアップはインストールコマンドがあるのでとても簡単です。
td-agentが使うRuby実行環境も、`/usr/lib/fluent/ruby`へインストールされるので、システムワイドに影響がなくて安心です。

次回はDashingで実際にジョブを書いてみようと思います。
このジョブでは、`Treasure Data Service`にクエリして、ダッシュボードのウィジェトを定期的に更新する予定です。





