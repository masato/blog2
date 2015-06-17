title: 'IDCFクラウド上のvagrant-lxcでGo開発環境'
date: 2014-05-16 21:32:47
tags:
 - vagrant-lxc
 - Go
 - IDCFクラウド
 - Martini
 - HelloWorld
description: この前作ったvagrant-lxcでGo開発環境を作ります。Nitrous.IOでN2Oを購入しようか悩んでいるところで、まずはIDCFクラウド上のVagrantで試してみます。3 Boxを使えるようにするには、$19.95/月かかります。もうちょっと安ければいいのに。
---
[この前](/2014/05/09/vagrant-on-idcf-64bit-lxc)作った[vagrant-lxc](https://github.com/fgrehm/vagrant-lxc)でGo開発環境を作ります。
Nitrous.IOでN2Oを[購入](https://www.nitrous.io/pricing)しようか悩んでいるところで、まずはIDCFクラウド上のVagrantで試してみます。
3 Boxを使えるようにするには、$19.95/月かかります。もうちょっと安ければいいのに。
<!-- more -->

### vagrant init

vagrant-lxcで使える[Box](https://github.com/fgrehm/vagrant-lxc-base-boxes)からTrustyを使います。

プロジェクトを作成します。
``` bash
$ mkdir -p ~/vagrant_apps/trusty64
$ cd !$
```

`vagrant init`
``` bash
$ vagrant init fgrehm/trusty64-lxc
```

Vagrantfileを編集します。
``` ruby ~/vagrant_apps/trusty64/Vagrantfile
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "fgrehm/trusty64-lxc"
  config.vm.network "forwarded_port", guest: 3000, host: 3000
end
```

### vagrant up

Vagrantを起動します。
``` bash
$ vagrant up --provider=lxc
Bringing machine 'default' up with 'lxc' provider...
==> default: Box 'fgrehm/trusty64-lxc' could not be found. Attempting to find and install...
    default: Box Provider: lxc
    default: Box Version: >= 0
==> default: Loading metadata for box 'fgrehm/trusty64-lxc'
    default: URL: https://vagrantcloud.com/fgrehm/trusty64-lxc
==> default: Adding box 'fgrehm/trusty64-lxc' (v1.1.0) for provider: lxc
    default: Downloading: https://vagrantcloud.com/fgrehm/trusty64-lxc/version/1/provider/lxc.box
==> default: Successfully added box 'fgrehm/trusty64-lxc' (v1.1.0) for 'lxc'!
==> default: Importing base box 'fgrehm/trusty64-lxc'...
==> default: Setting up mount entries for shared folders...
    default: /vagrant => /home/matedev/vagrant_apps/trusty64
==> default: Starting container...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 10.0.3.235:22
    default: SSH username: vagrant
    default: SSH auth method: private key
==> default: Machine booted and ready!
```

### ゲストOSにGoをインストール

BoxにSSH接続します。
``` bash
$ vagrant ssh
Welcome to Ubuntu 14.04 LTS (GNU/Linux 3.11.0-12-generic x86_64)

 * Documentation:  https://help.ubuntu.com/
vagrant@vagrant-base-trusty-amd64:~$
```

[gvm](https://github.com/moovweb/gvm)を使って、Go 1.2.2をインストールします。
``` bash
$ sudo apt-get update
$ sudo apt-get install golang curl git mercurial make binutils bison gcc build-essential
$ bash < <(curl -s -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer)
$ source /home/vagrant/.gvm/scripts/gvm
$ gvm install go1.2.2
$ gvm use go1.2.2
Now using version go1.2.2
```

インスト－ルされたGoを確認します。
``` bash
$ go version
go version go1.2.2 linux/amd64
$ which go
/home/vagrant/.gvm/gos/go1.2.2/bin/go
```

ソースからインストールすると、GOROOTとGOPATHをどうしようか悩みますが、
gvm経由だと適切に設定してくれます。
``` bash
$ echo $GOROOT
/home/vagrant/.gvm/gos/go1.2.2
$ echo $GOPATH
/home/vagrant/.gvm/pkgsets/go1.2.2/global
```

### Martiniのインストール

とりあえず、`Hello world`的に[Martini](https://github.com/go-martini/martini)を使ってみます。
``` bash
$ go get github.com/codegangsta/martini
```

プロジェクトの作成します。
``` bash
$ mkdir -p ~/go_apps/spike
$ cd !$
```

簡単なmain関数を書きます。
``` go ~/go_apps/spike/server.go
package main

import "github.com/codegangsta/martini"

func main() {
  m := martini.Classic()
  m.Get("/", func() string {
    return "Hello world!"
  })
  m.Run()
}
```

アプリケーションを実行します。
``` bash
$ go run server.go
[martini] listening on :3000 (development)
```

VagrantのホストOSから、curlで確認します。
``` bash
$ curl http://localhost:3000
Hello world!
```

### まとめ
簡単なGoのプログラムで、IDCFクラウドのインスタンスの上でvagrant-lxcが動作することが確認できました。
最近の[reddit](http://www.reddit.com/r/golang/comments/252wjh/are_you_using_golang_for_webapi_development_what/)を見ていたら、[Goji](https://goji.io/)が何となくいい感じなので、ちょっとしたプログラムを書いてみようと思います。


