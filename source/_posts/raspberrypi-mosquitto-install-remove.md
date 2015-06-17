title: "Raspberry PiにMosquittoのインストールとGPGキーの削除"
date: 2015-03-26 01:06:30
tags:
 - RaspberryPi
 - Mosquitto
 - Meshblu
description: Raspberry PiにMosquittoのクライアントをインストールしてMeshbluと直接MQTT通信してみます。MosquittoはRaspbianからデフォルトでapt-get installもできますが、mosquitto_pubに--helpフラグをつけてもバージョンが表示されないのでMosquitto Debian repositoryを使うことにします。
---

Raspberry PiにMosquittoのクライアントをインストールしてMeshbluと直接MQTT通信してみます。MosquittoはRaspbianからデフォルトで`apt-get install`もできますが、`mosquitto_pub`に`--help`フラグをつけてもバージョンが表示されないので`Mosquitto Debian repository`を使うことにします。

<!-- more -->

## インストール

[Mosquitto Debian repository](http://mosquitto.org/2013/01/mosquitto-debian-repository/)という記事に手順があるので従います。

``` bash
$ curl -O http://repo.mosquitto.org/debian/mosquitto-repo.gpg.key
$ sudo apt-key add mosquitto-repo.gpg.key
$ rm mosquitto-repo.gpg.key
$ cd /etc/apt/sources.list.d/
$ sudo curl -O http://repo.mosquitto.org/debian/mosquitto-repo.list
$ sudo apt-get update
```

MQTTブローカーはRaspberry Piに不要なのでMQTTクライアントのみインストールします。

``` bash
$ sudo apt-get install mosquitto-clients
```

`--help`フラグをつけてバージョンを確認します。

```
$ mosquitto_pub --help
mosquitto_pub is a simple mqtt client that will publish a message on a single topic and exit.
mosquitto_pub version 1.3.5 running on libmosquitto 1.3.5.
```

## パッケージとGPGキーの削除

一度きれいにパッケージを削除したかったのですが、GPGキーの削除方法がわからなかったので調べました。まずは普通にパッケージの削除と`sources.list`の削除をします。

``` bash
$ sudo apt-get remove --purge mosquitto-clients
$ sudo sudo rm /etc/apt/sources.list.d/mosquitto-repo.list
```

GPGキーの削除は`apt-key del`コマンドを使います。引数に`keyid`を指定しますが何がkeyidなのかよくわかりません。[Removing an unused GPG key?](http://forums.debian.net/viewtopic.php?f=10&t=52480)を参考にします。`apt-key list`を実行してMosquittoリポジトリを探します。`4096R/30993623`のスラッシュより後半部分が`keyid`のようです。

``` bash
$ sudo apt-key list
pub   4096R/30993623 2013-01-04 [expires: 2018-01-03]
uid                  Mosquitto Apt Repository <repo@mosquitto.org>
```

`apt-key del`に`keyid`を指定するとGPGキーが削除できました。`apt-key list`を実行してもMosquittoリポジトリが表示されなくなります。

```
$ sudo apt-key del 30993623
OK
```


