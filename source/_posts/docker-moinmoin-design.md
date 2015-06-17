title: "MoinMoinをDigitalOceanのCoreOSに移設する - Part1: はじめに"
date: 2014-10-12 01:40:39
tags:
 - MoinMoin
 - DigitalOcean
 - IDCFクラウド
 - CoreOS
 - fleet
 - flannel
 - Kubernetes
description: 来月にはDockerをプロダクション環境で使いたいので、まずは自分のサービスを移行してみます。さくらのVPSで2年半ほど運用しているMoinMoinを、DigitalOceanのCoreOSで動かすことをゴールにしました。現在はCentOS6.5 + Apache + mod_wsgiで動いていますが、Ubuntu + Nginx + uWSGIに移行します。コンテンツが増えて検索が遅くなってきたのでこの機会にXapianも使おうと思います。
---

* `Update 2014-10-13`: [MoinMoinをDigitalOceanのCoreOSに移設する - Part2: Dockerイメージの作成](/2014/10/13/docker-moinmoin-data-volume-container-nginx-uwsgi/)

来月にはDockerをプロダクション環境で使いたいので、まずは自分のサービスを移行してみます。さくらのVPSで2年半ほど運用しているMoinMoinを、DigitalOceanのCoreOSで動かすことをゴールにしました。

現在はCentOS6.5 + Apache + `mod_wsgi`で動いていますが、Ubuntu + Nginx + uWSGIに移行します。
コンテンツが増えて検索が遅くなってきたので、この機会に[Xapian](http://xapian.org/)も使おうと思います。

<!-- more -->

### DigitalOceanに移設する

Dockerホストの移設先は[CoreOSをサポートしたDigitalOcean](http://jp.techcrunch.com/2014/09/06/20140905digitalocean-partners-with-coreos-to-bring-large-scale-cluster-deployments-to-its-platform/)にしようと思います。

低価格なVPSでも[Kubernetesとflannel](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-kubernetes-on-top-of-a-coreos-cluster)でクラスタを組めばAWSやGCEで動かしているのと変わらなくなります。

### IDCFクラウドの開発環境とステージング環境

IDCFクラウド上で開発環境をUbuntuで構築して、ステージング環境はCoreOSで構築します。


### データボリュームコンテナ

データボリュームコンテナのバックアップとリストアは[手組みでtarとuntar](/2014/10/01/docker-wordpress-duplicate-volumes/)をしているので、[docker-backup](https://github.com/discordianfish/docker-backup)と[docker-lloyd](https://github.com/discordianfish/docker-lloyd)を試してみようと思います。
