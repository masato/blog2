title: 'Salt with Docker - Part2: WordPressインストール'
date: 2014-09-07 02:12:27
tags:
 - WordPress
 - MySQL
 - Salt
 - Docker
 - Tutum
 - Fig
description: SLSで構築したminionのDockerに、Docker Statesモジュールを使いWordPressコンテナを作成してみます。Saltには組み込みのStateモジュールたくさんあるので多くのことができますが、DockerでCMツールをどこまで使うべきなのか、見極めが問題です。
---

* `Update 2014-09-08`: [DockerにWordPress4.0日本語版を2-Container-Appインストール](/2014/09/08/docker-wordpress-2-container-app/)

[SLS](/2014/09/06/salt-idcf-docker-states/)で構築したminionのDockerに、[Docker States](http://docs.saltstack.com/en/latest/ref/states/all/salt.states.dockerio.html)モジュールを使いWordPressコンテナを作成してみます。

Saltには[組み込みのStateモジュール](http://docs.saltstack.com/en/latest/ref/states/all/index.html)たくさんあるので多くのことができますが、DockerでCMツールをどこまで使うべきなのか見極めが問題です。

<!-- more -->

### tutum/wordpress-stackableのイメージ

Dockerイメージは[Tutum](http://www.tutum.co/)の[tutum/wordpress-stackable](https://registry.hub.docker.com/u/tutum/wordpress-stackable/)を使います。WordPressとMySQLが別コンテナなので、MySQLは[tutum/mysql:5.5](https://registry.hub.docker.com/u/tutum/mysql/)を使います。

### WordPressコンテナのSLS

WordPressをインストールするためのSLSを作成します。SLSはYAML形式でFigっぽく書けます。
WordPressコンテナの起動前にMySQLコンテナが起動している必要があるため、statesの`require_in`がちょっと複雑になります。

``` yaml /srv/salt/wordpress/init.sls
mysql-image:
  docker.pulled:
    - name: tutum/mysql:5.5
    - require_in: mysql-container

mysql-container:
  docker.installed:
    - name: db
    - image: tutum/mysql:5.5
    - environment:
       - "MYSQL_PASS": "password"
    - require_in: mysql

mysql:
  docker.running:
    - container: db
    - port_bindings:
         "3360/tcp":
             HostIp: ""
             HostPort: "3360"
    - require_in: wordpress

wordpress-image:
  docker.pulled:
    - name: tutum/wordpress-stackable
    - require_in: wordpress-container

wordpress-container:
  docker.installed:
    - name: wordpress
    - image: tutum/wordpress-stackable
    - environment:
       - "DB_PASS": "password"
    - require_in: wordpress

wordpress:
  docker.running:
    - container: wordpress
    - links:
       db: db
    - port_bindings:
         "80/tcp":
             HostIp: ""
             HostPort: "80"
```

### Statesの実行

highstateでなく、作成したWordPressのStatesを指定してminion-wpに実行します。

``` bash
$ sudo salt 'minion-wp*' state.sls wordpress -v
...
Summary
------------
Succeeded: 6
Failed:    0
------------
Total:     6
```

### Figの場合

tutum/wordpress-stackableには、[Fig](https://github.com/tutumcloud/tutum-docker-wordpress-nosql/blob/master/fig.yml
)の設定ファイルも用意されています。SLSよりもシンプルにまとまります。

``` yaml fig.yml
wordpress:
  build: .
  links: 
   - db
  ports:
   - "80:80"
  environment:
    DB_NAME: wordpress
    DB_USER: admin
    DB_PASS: "**ChangeMe**"
    DB_HOST: "**LinkMe**"
    DB_PORT: "**LinkMe**"
db:
  image: tutum/mysql:5.5
  environment:
    MYSQL_PASS: "**ChangeMe**"
```

### 確認

minionのノードをブラウザから開いて確認します。WordPressのインストール画面が表示されました。

{% img center /2014/09/07/salt-idcf-docker-wordpress/wordpress.png %}


