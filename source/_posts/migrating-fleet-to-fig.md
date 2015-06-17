title: "CoreOSのfleetからfigに移行する - Part1: UbuntuとFigのインストール"
date: 2014-11-25 21:05:54
tags:
 - fig
 - Ubuntu
 - MoinMoin
description: VultrのCoreOSで動かしているMoinMoinですが、各unitが不定期にリスタートしてしまいます。だいたい3日間隔で。OpenResty、uWSGI、BitTorrent Syncのコンテナがどれもリスタートします。なぜかBitTorrent Syncを使っているData Volume Containerの同期がバックアップ先から巻き戻ってしまったので、数時間分の編集データが消えてしまいました。この機会にちょっとCoreOSから離れ、Figを使ったコンテナ管理を試してみようと思います。
---

VultrのCoreOSで動かしている[MoinMoin](https://registry.hub.docker.com/u/masato/moinmoin/)ですが、各unitが不定期にリスタートしてしまいます。だいたい3日間隔で。OpenResty、uWSGI、BitTorrent Syncのコンテナがどれもリスタートします。なぜかBitTorrent Syncを使っているData Volume Containerの同期がバックアップ先から巻き戻ってしまったので、数時間分の編集データが消えてしまいました。この機会にちょっとCoreOSから離れ、Figを使ったコンテナ管理を試してみようと思います。

<!-- more -->

### UbuntuにDockerをインストールする

[公式ドキュメント](http://docs.docker.com/installation/ubuntulinux/#ubuntu-trusty-1404-lts-64-bit)を参考にして、IDCFクラウドにUbuntu 14.04のインスタンスを作成します。

``` bash
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9
$ sudo sh -c "echo deb https://get.docker.io/ubuntu docker main > /etc/apt/sources.list.d/docker.list"
$ sudo apt-get update
$ sudo apt-get install lxc-docker
```

またはワンライナーでインストールもできます。

```
$ curl -sSL https://get.docker.com/ubuntu/ | sudo sh
```

### Figをインストールする

Figも[公式ドキュメント](http://www.fig.sh/install.html)を参考にしてインストールします。

``` bash
$ sudo -i
$ curl -L https://github.com/docker/fig/releases/download/1.0.1/fig-`uname -s`-`uname -m` > /usr/local/bin/fig; chmod +x /usr/local/bin/fig
```

バージョンを確認します。

``` bash
$ fig --version
fig 1.0.1
```
