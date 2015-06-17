title: 'Salt チュートリアル - Part4: Saltを使った監視インフラ'
date: 2014-09-17 00:44:09
tags:
 - Docker監視環境
 - Salt
 - Docker
 - MeasureAllTheThings
description: Saltを使いDockerホストをいくつか管理しています。それぞれ単独のホストなので連携はしていません。Dockerホストの監視はsalt-minionを監視のエージェントにも使ってみようという話しです。Saltにはstatsを取得できる組み込みモジュールがたくさん用意されています。また自分でPythonのモジュールも書けるので自由度は高いです。
---

* `Update 2014-09-26`: [Docker監視環境 - Part1: cAdvisor, collectd, Riemann, InfluxDB, Grafana](/2014/09/26/docker-monitoring-stack-prepare/)

Saltを使いDockerホストをいくつか管理しています。それぞれ単独のホストなので連携はしていません。
Dockerホストの監視はsalt-minionを監視のエージェントにも使ってみようという話しです。Saltにはstatsを取得できる[組み込みモジュール](http://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.ps.html#module-salt.modules.ps)がたくさん用意されています。また自分でPythonのモジュールも書けるので自由度は高いです。

<!-- more -->


### Monitoring Infrastructure

[Monitoring Infrastructure with SaltStack](https://speakerdeck.com/ipmb/monitoring-infrastructure-with-saltstack)のプレゼン資料に、salt-minionを監視エージェントとして使う各種メトリクスを集める方法が書いてあります。

もともと分散環境でZeroMQを使いPub-Subを行いリモートのコマンドを実行する仕組みなので、セキュアなネットワークとコマンドのスケジュール実行が可能なSaltは、監視インフラに向いています。

### Measure All the Things!

去年[Virtual Machine real time visualizations using cubism and salt](https://hveem.no/vm-monitoring-using-salt-and-cubism)を参考に分散システムのメトリクスをGraphiteに保存して、Cubism.jsでビジュアル化する開発を試みましたが、うまく行きませんでした。今回はDockerクラスタからメトリクスを集めるので、自分で`Measure of All the Things`ができそうです。PoC用のクラスタを自分の箱庭で作れるのでDockerは便利です。

やはりデータを欲しい人が自分でメトリクスを集めて、データを生成できる環境が必要です。
