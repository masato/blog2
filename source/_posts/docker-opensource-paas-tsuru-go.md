title: "DockerでオープンソースPaaS - Part7: tsuruでGoのHelloWorld"
date: 2014-07-08 23:14:37
tags:
 - DockerPaaS
 - tsuru
 - Go
 - HelloWorld
description: ようやくtsuruも少し仕組みがわかるようになってきました。標準ではPython用のイメージしかありませんが、tsuru-adminコマンドを使うとDockerfileを指定して、他のプログラム言語のプラットフォームのイメージを追加できます。basebuilderは、Dockerfileとデプロイスクリプトで構成されています。データベースを使うアプリを作る前に、Goで簡単な`Hello World`を書いて動作確認をします。
---

ようやくtsuruも少し仕組みがわかるようになってきました。標準ではPython用のイメージしかありませんが、
[tsuru-admin](http://docs.tsuru.io/en/0.5.2/apps/tsuru-admin/usage.html)コマンドを使うとDockerfileを指定して、他のプログラム言語のプラットフォームのイメージを追加できます。
[basebuilder](https://github.com/tsuru/basebuilder)は、Dockerfileとデプロイスクリプトで構成されています。

データベースを使うアプリを作る前に、Goで簡単な`Hello World`を書いて動作確認をします。

<!-- more -->
 

### Goプラットフォームの追加

[Go](http://docs.tsuru.io/en/0.5.2/apps/quickstart/go.html)アプリのサンプルがありますが、プラットフォームの追加方法がありません。

[basebuilder](https://github.com/tsuru/basebuilder)からGoのDockerファイルを使います。
 
``` https://raw.githubusercontent.com/tsuru/basebuilder/master/go/Dockerfile
# this file describes how to build tsuru go image
# to run it:
# 1- install docker
# 2- run: $ docker build -t tsuru/go https://raw.github.com/tsuru/basebuilder/master/go/Dockerfile

from	ubuntu:14.04
run	apt-get update
run	apt-get install wget -y --force-yes
run	wget http://github.com/tsuru/basebuilder/tarball/master -O basebuilder.tar.gz --no-check-certificate
run	mkdir /var/lib/tsuru
run	tar -xvf basebuilder.tar.gz -C /var/lib/tsuru --strip 1
run	cp /var/lib/tsuru/go/deploy /var/lib/tsuru
#run	cp /var/lib/tsuru/base/restart /var/lib/tsuru
run	cp /var/lib/tsuru/go/start /var/lib/tsuru
run	/var/lib/tsuru/go/install
run	/var/lib/tsuru/base/setup
```

platform-addコマンドでDockerfileのURLを指定してGoプラットフォームを追加します。

``` bash
$ tsuru-admin platform-add go -d https://raw.githubusercontent.com/tsuru/basebuilder/master/go/Dockerfile
```
 
### Goアプリの作成

tsuru上にGoアプリを作成します。helloworldという名前にしました。
`git push`するとデプロイ先のリポジトリが表示されます。

``` bash
$ tsuru app-create helloworld go
App "helloworld" is being created!
Use app-info to check the status of the app and its units.
Your repository for "helloworld" project is "git@10.1.1.17:helloworld.git"
```

`tsuru app-list`で確認をします。作成したhelloworldアプリはまだ起動していません。

``` bash
$ tsuru app-list
+-----------------+-------------------------+---------------------------+--------+
| Application     | Units State Summary     | Address                   | Ready? |
+-----------------+-------------------------+---------------------------+--------+
| helloworld      | 0 of 0 units in-service | helloworld.masato.pw      | Yes    |
| mysql-api       | 1 of 1 units in-service | mysql-api.masato.pw       | Yes    |
| tsuru-dashboard | 1 of 1 units in-service | tsuru-dashboard.masato.pw | Yes    |
+-----------------+-------------------------+---------------------------+--------+
```

### Goプロジェクト

プロジェクトを作成して、`git init`します。

``` bash
$ mkdir ~/go_apps/hello
$ cd !$
$ git init
Initialized empty Git repository in /home/mshimizu/go_apps/.git/
```

リクエストが来ると、`hello, world`を返す簡単なプログラムです。

``` go ~/go_apps/hello/main.go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", hello)
    fmt.Println("listening...")
    err := http.ListenAndServe(":8888", nil)
    if err != nil {
        panic(err)
    }
}

func hello(res http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(res, "hello, world")
}
```

変更を`git commit`します。

``` bash
$ git add .
$ git commit -m 'initial commit'
```

`app-info`でtsuruアプリの情報を確認できます。
TeamとOwnerはデフォルトのアカウントで作業しています。

``` bash
$ tsuru app-info -a helloworld
Application: helloworld
Repository: git@10.1.1.17:helloworld.git
Platform: go
Teams: admin
Address: helloworld.masato.pw
Owner: admin@example.com
Deploys: 0
```

デプロイ先のremoteリポジトリを追加します。

``` bash
$ git remote add tsuru git@10.1.1.17:helloworld.git
$ git push tsuru master
...
remote: ---- Starting 1 unit ----
remote:  ---> Started unit 1/1...
remote:
remote:  ---> App will be restarted, please check its logs for more details...
remote:
remote:
remote: OK
To git@10.1.1.17:helloworld.git
 * [new branch]      master -> master
```

### Procfileの作成

HerokuのようなProfileを作成して、Goアプリの起動コマンドを記述します。

``` yml ~/go_apps/hello/Proflile
web: ./main
```

Procfileの追加して`git push`するとデプロイが完了して、アプリが起動します。

``` bash
$ git add Procfile
$ git commit -m "Procfile: added file"
$ git push tsuru master
...
remote: ---- Starting 1 unit ----
remote:  ---> Started unit 1/1...
remote:
remote:  ---> App will be restarted, please check its logs for more details...
remote:
remote:
remote: OK
To git@10.1.1.17:helloworld.git
   696793b..0195778  master -> master
```

### 確認

`app-list`で確認すると、正常にデプロイがされ、アプリが開始しています。

``` bash
$ tsuru app-list
Application: helloworld
Repository: git@10.1.1.17:helloworld.git
Platform: go
Teams: admin
Address: helloworld.masato.pw
Owner: admin@example.com
Deploys: 2
Units:
+------------------------------------------------------------------+---------+
| Unit                                                             | State   |
+------------------------------------------------------------------+---------+
| ed5cd59d36b849e6770d32fc57506987ceb6e3df0de6c32094d219c69ae97f90 | started |
+------------------------------------------------------------------+---------+
```

curlで確認します。

``` bash
$ curl http://helloworld.masato.pw/
hello, world
```

ブラウザからの同様に表示されます。

http://helloworld.masato.pw/

### まとめ

一度言語のプラットフォームを作成してしまえば、簡単にGoアプリが動きます。
これはなかなかおもしろいです。気軽に使い捨てのアプリをデプロイできます。

アプリのデプロイと動作確認ができたので、最後はDjangoかRailsでデータベースサービスを利用した
アプリをtsuruにデプロイしてみようと思います。




