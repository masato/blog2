title: "BlueMixでHello Go"
date: 2014-06-25 23:21:15
tags:
 - Bluemix
 - Go
 - CloudFoundry
 - HelloWorld
---

[IBM Bluemix Challenge](http://www.ibm.com/developerworks/jp/bluemix/contest/index.html)が始まっていたので遅くなりましたがようやくエントリしました。

オフィシャル以外にも`Cloud Foundry`用に各言語のBuildpackがいくつかあるので、遊びながら何かネタを考えたいと思います。
なんと`BLU Acceleration`も[使える](http://www.ibm.com/developerworks/jp/bigdata/library/bd-ruby-bluacceleration-app/)みたいです。

それにしても最近の[developerWorks](http://www.ibm.com/developerworks/jp/views/cloud/libraryview.jsp)はおもしろいです。DigitalOceanの[Tutorials](https://www.digitalocean.com/community/tutorials)とかも、写経していて楽しいです。

BlueMixはDojoのようなので、Multi-LevelPushMenuとは違うようですが、
左メニューは最近お気に入りの多階層ナビゲーションです。SoftLayer買収後からかなり印象が変わりました。

まずはGoのBuildpackを使ってみて、うまく動いたらDartも試してみようと思います。

<!-- more -->

BlueMixはIBMがSolftLayer上に構築した、`Cloud Foundry`ベースのPaaSです。
特徴的なのは、`IBM DevOps Services (旧JazzHub)`と連携すると、 RationalのJazzとWebIDEのOrionが使えます。

Nitrous.IOやMonacoのような環境が気に入っているので、WebIDEは積極的に使いたいです。

## GoのBuildpackと開発環境

GoのBuildpackは[michaljemala/cloudfoundry-buildpack-go](https://github.com/michaljemala/cloudfoundry-buildpack-go)を使います。
GitHubのREADMEにサンプルコードがあるので、これに従って進めていきます。

開発環境は、昨日できたての作業ユーザーを作った[Dockerfile](/2014/06/24/docker-devenv-adduser-add-permission-denied/)を使います。

``` bash
$ docker run --rm -i -t masato/baseimage:1.8 /sbin/my_init /bin/bash
# su - docker
```

Git、Ruby、Emacs、byobuと基本の開発環境が一瞬で起動すると、とても気分がよいです。

## michaljemala/hello-goをclone

サンプルの[hell-go](https://github.com/michaljemala/hello-go.git)を`git clone`します。

``` bash
$ mkdir ~/cf_apps
$ cd !$
$ git clone https://github.com/michaljemala/hello-go.git
```

`Cloud Foundry`のデプロイ設定であるmanifest.ymlを自分用に編集します。
nameとhostを自分用に変更しました。メモリのサイズや起動するインスタンス数を指定します。

``` yml ~/cf_apps/hello-go/manifest.yml
applications:
- name: masato-hello-go
  memory: 8M
  instances: 1
  buildpack: https://github.com/michaljemala/cloudfoundry-buildpack-go.git
  host: masato-hello-go
  domain: mybluemix.net
  path: .
```

修正するファイルはこれだけです。プロジェクトは、Goのプログラムファイル、.godir、Procfileで構成されています。

``` bash
$ tree hello-go -a -L 1
hello-go
├── .git
├── .godir
├── Procfile
├── README.md
├── manifest.yml
└── web.go

1 directory, 5 files
```

Procfileはforemanが起動するプロセスを記述するファイルです。

``` ruby ~/cf_apps/hello-go/Procfile
web: hello-go
```

.godirはよくわかりませんが、アプリケーションの名前を書くようです。

``` ruby ~/cf_apps/hello-go/.godir
hello-go
```

8080でLISTENするGoのサーバープログラムです。
`/`のリクエストハンドラで、VCAP環境変数などの実行環境の情報をダンプします。

``` go ~/cf_apps/hello-go/web.go
package main

import (
        "code.google.com/p/log4go"
        "fmt"
        "launchpad.net/goyaml"
        "net/http"
        "os"
)

const (
        HostVar = "VCAP_APP_HOST"
        PortVar = "VCAP_APP_PORT"
)

type T struct {
        A string
        B []int
}

func main() {
        log := make(log4go.Logger)
        log.AddFilter("stdout", log4go.DEBUG, log4go.NewConsoleLogWriter())

        http.HandleFunc("/", hello)
        var port string
        if port = os.Getenv(PortVar); port == "" {
                port = "8080"
        }
        log.Debug("Listening at port %v\n", port)
        if err := http.ListenAndServe(":"+port, nil); err != nil {
                panic(err)
        }
}

func hello(res http.ResponseWriter, req *http.Request) {
        // Dump ENV
        fmt.Fprint(res, "ENV:\n")
        env := os.Environ()
        for _, e := range env {
                fmt.Fprintln(res, e)
        }
        fmt.Fprint(res, "\nYAML:\n")

        //Dump some YAML
        t := T{A: "Foo", B: []int{1, 2, 3}}
        if d, err := goyaml.Marshal(&t); err != nil {
                fmt.Fprintf(res, "Unable to dump YAML")
        } else {
                fmt.Fprintf(res, "--- t dump:\n%s\n\n", string(d))
        }
}
```

### cfツールのインストール

`Cloud Foundry`は実装をGoに移行しているようで、まずCLIのcfコマンドがGoの実装に変更されています。
Goらしく、バイナリをPATHが通っているところに配置するだけでインストールは終了です。

``` bash
$ mkdir ~/bin
$ wget https://s3.amazonaws.com/go-cli/releases/v6.1.2/cf-linux-amd64.tgz
$ tar zxvf cf-linux-amd64.tgz  -C ~/bin
$ chmod u+x ~/bin/cf
$ source .profile
$ which cf
/home/docker/bin/cf
$ cf --version
cf version 6.1.2-6a013ca
```

cfコマンドを使いBlueMixのエンドポイントを指定してログインします。

``` bash
$ cf login

API endpoint> https://api.ng.bluemix.net
Email> {メールアドレス}

Password> {パスワード}
Authenticating...
OK
```

### BlueMixへPUSH 

Buildpackを指定してPUSHします。プロジェクトにあるmanifest.ymlを使います。

``` bash
$ cf push masato-hello-go --b https://github.com/michaljemala/cloudfoundry-buildpack-go.git
Using manifest file /home/docker/go_apps/hello-go/manifest.yml
...
-----> Downloaded app package (4.0K)
Cloning into '/tmp/buildpacks/cloudfoundry-buildpack-go'...
OK
 Installing Go 1.2.1...  done in 7s
 Installing Virtualenv...  done
 Downloading Mercurial sources...  done in 4s
 Installing Mercurial...  done
 Downloading Bazaar sources...  done in 15s
 Installing Bazaar...  done
 Running 'go get -tags cf ./...'...  done in 45s
-----> Uploading droplet (2.0M)
...
requested state: started
instances: 1/1
usage: 8M x 1 instances
urls: masato-hello-go.mybluemix.net

     state   since                    cpu    memory   disk
#0   down    2014-06-26 02:27:59 AM   0.0%   0 of 0   0 of 0
```

### 確認

curlコマンドの-Iオプションで、HTTPヘッダを表示します。
curlコマンドでアプリが返す、実行環境の情報を表示できます。

``` bash
$ curl -I http://masato-hello-go.mybluemix.net
HTTP/1.1 200 OK
...
$ curl -I http://masato-hello-go.mybluemix.net
ENV:
TMPDIR=/home/vcap/tmp
VCAP_APP_PORT=62536
USER=vcap
...
```

### まとめ

`VisualStudio Online`も含め、PaaSとGitリポジトリ、プロジェクト管理ツールやコミュニケーションツール、
WebIDEが一体になったクラウドサービスが出始めています。

PivotalやSAP、IBMもデザインシンキングを提唱して、今年はエンタープライズ向けの、
クラウド上でのアジャイルな開発環境やテスト環境が本格的化しそうな雰囲気です。

個人的にはNitrous.IOみたいにコンパクトな開発環境が好みですが。

次回はBlueMixの特徴である、`IBM DevOps Services`と連携させて、Gitリポジトリや、WebIDEのOrionを試してみます。
本当にChromeだけで開発環境が完結できそうな気がしていきました。





