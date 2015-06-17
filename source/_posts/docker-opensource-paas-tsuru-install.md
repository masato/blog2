title: "DockerでオープンソースPaaS - Part3: tsuruインストール"
date: 2014-07-04 23:44:52
tags:
 - Docker
 - DockerPaaS
 - tsuru
 - IDCFクラウド
description: ようやく時間ができたので、さっそくtsuruをIDCFクラウドにインストールしてみます。流行のワンライナーインストールのTsuru Nowがあるのでこれを使います。tsuruのドキュメントはちょっとわかりづらいです。ワンライナーでも何か問題に遭遇するとインストール手順は自分で理解して解決する必要があります。
---

ようやく時間ができたので、さっそくtsuruをIDCFクラウドにインストールしてみます。
流行のワンライナーインストールの[Tsuru Now](https://github.com/tsuru/now)があるのでこれを使います。

tsuruの[ドキュメント](http://tsuru.readthedocs.org/en/latest/)はちょっとわかりづらいです。
ワンライナーでも何か問題に遭遇するとインストール手順は自分で理解して解決する必要があります。

<!-- more -->
 
### 環境の用意

[ドキュメント](http://docs.tsuru.io/en/latest/docker.html)はUbuntu 14.04を想定しているのですが、
IDCFクラウドでは標準で提供されていないので、[ISOをダウンロード](http://ubuntutym2.u-toyama.ac.jp/ubuntu/trusty/ubuntu-14.04-server-amd64.iso)してインストールして使います。

Pythonのヘッダが必要な気がするので、最初にインストールしておきます。

``` bash
$ sudo apt-get install python-dev
```

### Tsuru Nowの実行

[Tsuru Now](https://github.com/tsuru/now)を使うとワンライナーでインストールできます。

``` bash
$ curl -sL https://raw.github.com/tsuru/now/master/run.bash | bash
```

ただREADMEを読んでもオプションの渡し方がわかないので、スクリプトを一度ダウンロードしてから実行します。

``` bash
$ curl -sL https://raw.github.com/tsuru/now/master/run.bash -o run.bash
```

また日本語環境だと`inet addr`のところがgrepできないので、`LANG=C`を指定します。

``` bash　 run.bash 
    if [[ $host_ip == "" ]]; then
        host_ip=$(ifconfig | grep -A1 eth | grep "inet addr" | tail -n1 | sed "s/[^0-9]*\([0-9.]*\).*/\1/")
    fi
```

インストールを実行します。`--host-name`オプションで[昨日](/2014/07/03/docker-opensource-paas-httprouting/)Namecheapで購入したドメイン名を指定します。

``` bash
$ LANG=C bash ./run.bash --host-name {ドメイン名}.pw
...
######################## DONE! ########################

Some information about your tsuru installation:

Admin user: admin@example.com
Admin password: admin123 (PLEASE CHANGE RUNNING: tsuru change-password)
Target address: 10.1.1.17:8080
Dashboard address: 10.1.1.17:

To use Tsuru router you should have a DNS entry *.masato.pw -> 10.1.1.17

You should run `source ~/.bashrc` on your current terminal.

Installed apps:
+-----------------+-------------------------+---------------------------+--------+
| Application     | Units State Summary     | Address                   | Ready? |
+-----------------+-------------------------+---------------------------+--------+
| tsuru-dashboard | 1 of 1 units in-service | tsuru-dashboard.masato.pw | Yes    |
+-----------------+-------------------------+---------------------------+--------+
```

### ユーザーの作成

インストールが終了したら、ログインユーザーを作成します。

``` bash
$ tsuru user-create ma6ato@gmail.com
Password:
Confirm:
User "ma6ato@gmail.com" successfully created!
```

### インストールの確認

tsuru-dashboardをブラウザから確認します。

NamecheapのワイルドカードDNSと、Hipacheの`HTTP Routing`のおかげで、
Dockerコンテナのアプリに簡単にルーティングができます。

http://tsuru-dashboard.masato.pw/

このあと、さきほど作成したユーザーでログインすると簡単なダッシュボードを使えるようになります。

{% img center /2014/07/04/docker-opensource-paas-tsuru-install/tsuru-dashboard.png %}


### まとめ

インストール自体はワンライナーで簡単なのですが、何らかの問題でインストールが止まってしまったり、
設定をやり直すため、再インストールが必要な場合があります。

次回はtsuruの再インストール方法を確認しながら、削除しても繰り返し構築できるようにします。
