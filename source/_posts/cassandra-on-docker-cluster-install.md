title: "Cassandra on DockerでIoT用データストアを用意する - Part2: Clusterインストール"
date: 2015-01-20 20:10:23
tags:
 - Cassandra
 - IoT
 - Spotify
 - Docker
 - cqlsh
 - CoreOS
description: シングルノードのCassandraをDockerコンテナで起動確認できました。次にspotify/cassandra clusterイメージを使いCassandraクラスタを構築して、簡単なkeyspaceとtableを作成してみます。
---

[シングルノードのCassandra](/2015/01/19/cassandra-on-docker-install/)をDockerコンテナで起動確認できました。次に[spotify/cassandra:cluster](https://registry.hub.docker.com/u/spotify/cassandra/)イメージを使いCassandraクラスタを構築して、簡単なkeyspaceとtableを作成してみます。

<!-- more -->

## CoreOSにボリューム用ディレクトリを作成

Cassandraコンテナにマウントするボリューム用のディレクトリを作成します。CoreOSの場合はmkdirで作成できるディレクトリに制限があります。cloud-configの`write_files`ディレクティブに3ノード用のディレクトリを作成します。

``` yaml:/var/lib/coreos-install/user_data
#cloud-config
write_files:
...
  - path: /var/lib/cassandra/c1/.README
    owner: core:core
    permissions: 0644
    content: |
      Cassandra cluster c1 directory
  - path: /var/lib/cassandra/c2/.README
    owner: core:core
    permissions: 0644
    content: |
      Cassandra cluster c2 directory
  - path: /var/lib/cassandra/c3/.README
    owner: core:core
    permissions: 0644
    content: |
      Cassandra cluster c3 directory
```

coreos-cloudinitの再実行をします。

``` bash
$ sudo coreos-cloudinit --from-file /var/lib/coreos-install/user_data
```


## トークンの計算

今回は3ノード構成のクラスタを構築するので、最初に[トークンを計算して生成](http://www.datastax.com/documentation/cassandra/2.0/cassandra/configuration/configGenTokens_c.html)します。

``` bash
$ python -c 'print [str(((2**64 / 3) * i) - 2**63) for i in range(3)]'
['-9223372036854775808', '-3074457345618258603', '3074457345618258602']
```

## 1台目のノード

1台目はシードノードにします。環境変数`CASSANDRA_SEEDS`を指定しないと、インストール時にコンテナのIPアドレスを使います。環境変数`CASSANDRA_TOKEN`は先ほど計算したトークンを順番に使っていきます。

``` bash
$ docker pull spotify/cassandra:cluster
$ docker run -d -v /var/lib/cassandra/c1:/var/lib/cassandra \
  -e "CASSANDRA_TOKEN=-9223372036854775808" \
  --name c1 \
  spotify/cassandra:cluster
```

c1コンテナのbashを起動します。

``` bash
$ docker exec -it c1 /bin/bash
```

シードノードのIPアドレスを確認します。このIPアドレスは他の2ノードの起動時に環境変数`CASSANDRA_SEEDS`として使用します。

``` bash
$ hostname --ip-address | cut -f 1 -d ' '
172.17.0.6
```

netstatでバインドしているIPアドレスを確認します。

``` bash
$ netstat -alnp | grep 9160
tcp        0      0 172.17.0.6:9160         0.0.0.0:*               LISTEN      16/java
```

IPアドレスを指定してcqlshの動作確認をします。

``` bash
$ cqlsh 172.17.0.6
Connected to Test Cluster at 172.17.0.6:9160.
[cqlsh 4.1.1 | Cassandra 2.0.10 | CQL spec 3.1.1 | Thrift protocol 19.39.0]
Use HELP for help.
cqlsh>
```

## 2台目と3台目のノード

2-3台目はシードノードを環境変数`CASSANDRA_SEEDS`に指定して起動します。環境変数`CASSANDRA_TOKEN`も1台目と同様に順番に使っていきます。まず2台目のノードを起動します。

``` bash
$ docker run -d -v /var/lib/cassandra/c2:/var/lib/cassandra \
  -e "CASSANDRA_SEEDS=172.17.0.6" \
  -e "CASSANDRA_TOKEN=-3074457345618258603" \
  --name c2 \
  spotify/cassandra:cluster
```

3台目のノードを起動します。

``` bash
$ docker run -d -v /var/lib/cassandra/c3:/var/lib/cassandra \
  -e "CASSANDRA_SEEDS=172.17.0.6" \
  -e "CASSANDRA_TOKEN=3074457345618258602" \
  --name c3 \
  spotify/cassandra:cluster
```

## クラスタの確認

Cassandraクラスタ用のコンテナが3台起動しました。

``` bash
$ docker ps|head
CONTAINER ID        IMAGE                       COMMAND                CREATED             STATUS              PORTS                                                                           NAMES
7501f4754cea        spotify/cassandra:cluster   "cassandra-clusterno   4 seconds ago       Up 4 seconds        9160/tcp, 22/tcp, 61621/tcp, 7000/tcp, 7001/tcp, 7199/tcp, 8012/tcp, 9042/tcp   c3
2a9fe6fc4a1b        spotify/cassandra:cluster   "cassandra-clusterno   25 seconds ago      Up 25 seconds       9160/tcp, 22/tcp, 61621/tcp, 7000/tcp, 7001/tcp, 7199/tcp, 8012/tcp, 9042/tcp   c2
aa5335d223bd        spotify/cassandra:cluster   "cassandra-clusterno   3 minutes ago       Up 3 minutes        7000/tcp, 7001/tcp, 7199/tcp, 8012/tcp, 9042/tcp, 9160/tcp, 22/tcp, 61621/tcp   c1
```

c1ノードのbashを起動して、ノードのステータスを表示します。

``` bash
$ docker exec -it c1 /bin/bash
$ nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens  Owns (effective)  Host ID                               Rack
UN  172.17.0.8  64.8 KB    1       50.0%             23df3501-5abe-4b1e-9e26-f24aeb7774a9  rack1
UN  172.17.0.7  115.59 KB  1       50.0%             ee3e867e-f292-444f-90eb-95e3b699d138  rack1
UN  172.17.0.6  124.99 KB  1       50.0%             4def9a7d-90e6-45a3-9752-07305a5e71e1  rack1
DN  172.17.0.5  ?          1       50.0%             45095f6d-490e-48cc-b1c6-c1bfc19d6eb1  rack1
```

fleetctlだと便利ですが、まだunitファイルを作成していないので、各ノードのIPアドレスをそれぞれ確認します。

```
$ docker inspect -f '&#123;&#123; .NetworkSettings.IPAddress }}' c1
172.17.0.6
$ docker inspect -f '&#123;&#123; .NetworkSettings.IPAddress }}' c2
172.17.0.7
$ docker inspect -f '&#123;&#123; .NetworkSettings.IPAddress }}' c3
172.17.0.8
```

## keyspaceとtableを作成してクラスタの動作確認

c1ノード上で作業します。

``` bash
$ docker exec -it c1 /bin/bash
```

keyspaceとtableの作成、初期データ投入用のスクリプトを記述します。`replication_factor`に2を指定しているので、2ノードに冗長化されます。

``` cql:~/create_keyspace.cql
CREATE KEYSPACE test WITH REPLICATION =
 {'class': 'SimpleStrategy', 'replication_factor': 2};

 USE test;

 CREATE TABLE test_table (
  id text,
  test_value text,
  PRIMARY KEY (id)
 );


INSERT INTO test_table (id, test_value) VALUES ('1', 'one');
INSERT INTO test_table (id, test_value) VALUES ('2', 'two');
INSERT INTO test_table (id, test_value) VALUES ('3', 'three');
```

作成したCQLファイルを実行します。

``` bash
$ cqlsh -f create_keyspace.cql 172.17.0.6
```

c1ノードからcqlshコマンドを起動します。クエリを実行してデータ取得ができました。

``` bash
$ docker exec -it c1 /bin/bash
$ cqlsh 172.17.0.6
cqlsh> use test;
cqlsh:test> SELECT * FROM test_table;

 id | test_value
----+------------
  3 |      three
  2 |        two
  1 |        one

(3 rows)

cqlsh:test>
```

今度は別のc2ノードからクエリをテストします。

``` bash
$ docker exec -it c2 /bin/bash
$ cqlsh 172.17.0.7
Connected to Test Cluster at 172.17.0.7:9160.
[cqlsh 4.1.1 | Cassandra 2.0.10 | CQL spec 3.1.1 | Thrift protocol 19.39.0]
Use HELP for help.
cqlsh> use test;
cqlsh:test> SELECT * FROM test_table;

 id | test_value
----+------------
  3 |      three
  2 |        two
  1 |        one

(3 rows)

cqlsh:test>
```


