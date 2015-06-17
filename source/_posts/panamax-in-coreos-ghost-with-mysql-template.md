title: "Panamax で Dockerコンテナ管理- Part2: Ghost with MySQL"
date: 2014-08-29 01:31:33
tags:
 - Docker管理
 - Panamax
 - CoreOS
 - Ghost
 - MySQL
description: IDCFクラウドのCoreOSにインストールしたPanamaxへ、Templateを使ってDockerコンテナをデプロイしてみます。App Template Challengeが9月2日まで開催されていることもあり、たくさんTemplateが見つかります。とりあえず適当にRailsのTemplateを探してRun Templateしたらどれも動かなかったので、以下の基準でTemplateを探しました。Panamax専用のイメージを使っていないこと、アプリとDBコンテナが分かれていること、IDCFクラウドで動かしているため、Vagrant固有の機能を使っていないこと
---

[IDCFクラウドのCoreOSにインストールした](/2014/08/19/panamax-in-coreos-on-idcf/)Panamaxへ、Templateを使ってDockerコンテナをデプロイしてみます。[App Template Challenge](http://panamax.io/contest/)が9月2日まで開催されていることもあり、たくさんTemplateが見つかります。
とりあえず適当にRailsのTemplateを探して`Run Template`したらどれも動かなかったので、以下の基準でTemplateを探しました。

* Panamax専用のイメージを使っていないこと
* アプリとDBコンテナが分かれていること
* IDCFクラウドで動かしているため、Vagrant固有の機能を使っていないこと

<!-- more -->

### Templateの定義ファイル

ブログツールのGhostかWagtailを使ってみよう思い、まずGhostのpmxファイルを[ghost_with_mysql.pmx](https://github.com/CenturyLinkLabs/panamax-contest-templates/blob/master/ghost_with_mysql.pmx)でGitHubで確認します。

Figの書式に似ているので、[Building Complex Apps for Docker on CoreOS and Fig](http://www.centurylinklabs.com/building-complex-apps-for-docker-on-coreos-and-fig/)で紹介されていた[fig2coreos](https://github.com/centurylinklabs/fig2coreos)が発展した感じです。

PanamaxはCoreOS上で動くため、orchestrationはfleetを使います。Figを開発したOrchardはCoreOSに買収されたので、このYAMLからfleet用のsystemdのunitファイルを生成する方法は標準的になりそうです。fleetはDockerコンテナ管理のためによくできた仕組みです。


``` yaml ghost_with_mysql.pmx
...
images:
- name: dockerfile_ghost_latest
  source: dockerfile/ghost:latest
  type: Default
  ports:
  - host_port: '8080'
    container_port: '2368'
    proto: TCP
  links:
  - service: centurylink_mysql
    alias: centurylink_mysql
  environment:
  - variable: DB_PASSWORD
    value: pass@word01
  - variable: DB_NAME
    value: ghost
- name: centurylink_mysql
  source: centurylink/mysql:5.5
  category: DB Tier
  type: Default
  ports:
  - host_port: '3306'
    container_port: '3306'
    proto: TCP
  environment:
  - variable: MYSQL_ROOT_PASSWORD
    value: pass@word01
```

どちらも汎用的なDockerイメージを使っているのでテストに向いていそうです。

* MySQL: [docker-mysql](https://github.com/CenturyLinkLabs/docker-mysql)
* Ghost: [dockerfile/ghost](https://github.com/dockerfile/ghost)

### 環境変数

PanamaxのTemplateは起動する前に環境変数が正常に設定されていないと、[Infinite loop trying to add official default MySQL to empty application](https://github.com/CenturyLinkLabs/panamax-ui/issues/286)のようにsytemdが無限ループします。Panamaxを始めて使うと動かないので戸惑います。

今回のGhostの環境変数は、MySQL周辺ですが初期値が正常に入っているのでこのまま動きそうです。

### Run Templates

Searchタブから`Ghost with MySQL`を検索して`Run Templates`ボタンを押すだけで、イメージ取得とfleetのunitファイル作成、アプリケーション起動、コンテナ連携をしてくれます。

`CoreOS Journal - Application Activity Log`で、systemdのjournalからログを確認できます。以下は`DB Tier`の、centurylink_mysqlコンテナが起動したところまでです。

``` bash
Aug 28 10:36:18 systemd Starting centurylink_mysql.service...
Aug 28 10:36:18 docker Pulling repository centurylink/mysql
Aug 28 10:37:42 systemd Started centurylink_mysql.service.
Aug 28 10:37:42 systemd Starting dockerfile_ghost_latest.service...
Aug 28 10:37:42 docker Error response from daemon: No such container: centurylink_mysql
Aug 28 10:37:42 docker 2014/08/28 01:37:42 Error: failed to remove one or more containers
Aug 28 10:37:42 docker Pulling repository dockerfile/ghost
Aug 28 10:37:42 docker 140828 1:37:42 [Warning] Using unique option prefix key_buffer instead of key_buffer_size is deprecated and will be removed in a future release. Please use the full name instead.
Aug 28 10:37:42 docker 140828 1:37:42 [Warning] Using unique option prefix key_buffer instead of key_buffer_size is deprecated and will be removed in a future release. Please use the full name instead.
Aug 28 10:37:43 docker 140828 1:37:43 [Warning] Using unique option prefix key_buffer instead of key_buffer_size is deprecated and will be removed in a future release. Please use the full name instead.
Aug 28 10:37:43 docker 140828 1:37:43 [Warning] Using unique option prefix myisam-recover instead of myisam-recover-options is deprecated and will be removed in a future release. Please use the full name instead.
Aug 28 10:37:43 docker 140828 1:37:43 [Note] Plugin 'FEDERATED' is disabled.
Aug 28 10:37:43 docker 140828 1:37:43 InnoDB: The InnoDB memory heap is disabled
Aug 28 10:37:43 docker 140828 1:37:43 InnoDB: Mutexes and rw_locks use GCC atomic builtins
Aug 28 10:37:43 docker 140828 1:37:43 InnoDB: Compressed tables use zlib 1.2.8
Aug 28 10:37:43 docker 140828 1:37:43 InnoDB: Using Linux native AIO
Aug 28 10:37:43 docker 140828 1:37:43 InnoDB: Initializing buffer pool, size = 128.0M
Aug 28 10:37:43 docker 140828 1:37:43 InnoDB: Completed initialization of buffer pool
Aug 28 10:37:43 docker 140828 1:37:43 InnoDB: highest supported file format is Barracuda.
Aug 28 10:37:43 docker 140828 1:37:43 InnoDB: Waiting for the background threads to start
Aug 28 10:37:44 docker 140828 1:37:44 InnoDB: 5.5.38 started; log sequence number 1595675
Aug 28 10:37:44 docker 140828 1:37:44 InnoDB: Starting shutdown...
Aug 28 10:37:45 docker 140828 1:37:45 InnoDB: Shutdown completed; log sequence number 1595675
Aug 28 10:37:45 docker 140828 1:37:45 [Warning] Using unique option prefix key_buffer instead of key_buffer_size is deprecated and will be removed in a future release. Please use the full name instead.
Aug 28 10:37:45 docker 140828 1:37:45 [Warning] Using unique option prefix myisam-recover instead of myisam-recover-options is deprecated and will be removed in a future release. Please use the full name instead.
Aug 28 10:37:45 docker 140828 1:37:45 [Note] Plugin 'FEDERATED' is disabled.
Aug 28 10:37:45 docker 140828 1:37:45 InnoDB: The InnoDB memory heap is disabled
Aug 28 10:37:45 docker 140828 1:37:45 InnoDB: Mutexes and rw_locks use GCC atomic builtins
Aug 28 10:37:45 docker 140828 1:37:45 InnoDB: Compressed tables use zlib 1.2.8
Aug 28 10:37:45 docker 140828 1:37:45 InnoDB: Using Linux native AIO
Aug 28 10:37:45 docker 140828 1:37:45 InnoDB: Initializing buffer pool, size = 128.0M
Aug 28 10:37:45 docker 140828 1:37:45 InnoDB: Completed initialization of buffer pool
Aug 28 10:37:45 docker 140828 1:37:45 InnoDB: highest supported file format is Barracuda.
Aug 28 10:37:45 docker 140828 1:37:45 InnoDB: Waiting for the background threads to start
Aug 28 10:37:46 docker 140828 1:37:46 InnoDB: 5.5.38 started; log sequence number 1595675
Aug 28 10:37:46 docker 140828 1:37:46 [Note] Server hostname (bind-address): '0.0.0.0'; port: 3306
Aug 28 10:37:46 docker 140828 1:37:46 [Note] - '0.0.0.0' resolves to '0.0.0.0';
Aug 28 10:37:46 docker 140828 1:37:46 [Note] Server socket created on IP: '0.0.0.0'.
Aug 28 10:37:46 docker 140828 1:37:46 [Note] Event Scheduler: Loaded 0 events
Aug 28 10:37:46 docker 140828 1:37:46 [Note] /usr/sbin/mysqld: ready for connections.
Aug 28 10:37:46 docker Version: '5.5.38-0ubuntu0.14.04.1-log' socket: '/var/run/mysqld/mysqld.sock' port: 3306 (Ubuntu)
```

正常にTemplateからコンテナが2台起動しました。

{% img center /2014/08/29/panamax-in-coreos-ghost-with-mysql-template/ghost_app.png %}

### コンテナの起動確認

Panamaxが動いているDockerホストのCoreOSにSSH接続します。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh -A core@10.1.0.66
```

起動したコンテナのプロセスを確認します。

``` bash
$ docker ps
CONTAINER ID        IMAGE                            COMMAND                CREATED             STATUS              PORTS                     NAMES
...
6a60a060dd7f        dockerfile/ghost:latest          bash /ghost-start      22 hours ago        Up 22 hours         0.0.0.0:8080->2368/tcp    dockerfile_ghost_latest
40526d34a765        centurylink/mysql:5.5            /usr/local/bin/run     22 hours ago        Up 22 hours         0.0.0.0:3306->3306/tcp    centurylink_mysql,dockerfile_ghost_latest/centurylink_mysql
...
```

MySQLコンテナをinspectするとpmxで設定した環境変数が設定されています。

``` bash
$ docker inspect --format="&#123;&#123; .Config.Env &#125;&#125;" 40526d34a765
[MYSQL_ROOT_PASSWORD=pass@word01 HOME=/ PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin DEBIAN_FRONTEND=noninteractive]
```

### Ghostをブラウザで確認

`docker ps`でみると、Panamaxを動かしているCoreOSに8080でポートがマップされています。
Dockerホストから直接CoreOSのIPアドレスをブラウザで確認します。


{% img center /2014/08/29/panamax-in-coreos-ghost-with-mysql-template/ghost_demo.png %}

