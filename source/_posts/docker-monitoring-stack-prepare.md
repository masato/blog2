title: "Docker監視環境 - Part1: cAdvisor, collectd, Riemann, InfluxDB, Grafana"
date: 2014-09-26 00:46:55
tags:
 - Docker監視環境
 - MeasureAllTheThings
 - Salt
 - cAdvisor
 - collectd
 - Riemann
 - InfluxDB
 - Grafana
 - StreamProcessing
description: いままで仮想マシンで動かしていた本番のシステムをDockerコンテナに移設しようと思います。KubernetesとかDeisでDockerコンテナを管理するか悩むところですが、とりあえずはSaltでDockerホストをCMしながらプラクティスです。これまでうまく実現できなかったMeasure All the Thingsとメトリクス収集もDockerならOpsに依頼しなくても自分でハンドルできそうなので、RiemannやLaminaなどのストリームデータのリアルタイム分析もClojureで試してみようと思います。
---

* `Update 2014-10-08`: [Docker監視環境 - Part3: cAdvisor,InfluxDB,Grafanaの構築](/2014/10/08/docker-monitoring-stack-cadvisor-influxdb-grafana/)
* `Update 2014-10-07`: [Docker監視環境 - Part2: Riemannの構築](/2014/10/07/docker-monitoring-stack-riemann)


いままで仮想マシンで動かしていた本番のシステムをDockerコンテナに移設しようと思います。KubernetesとかDeisでDockerコンテナを管理するか悩むところですが、とりあえずはSaltでDockerホストをCMしながらプラクティスです。
これまでうまく実現できなかった`Measure All the Things`とメトリクス収集もDockerならOpsに依頼しなくても自分でハンドルできそうなので、RiemannやLaminaなどのストリームデータのリアルタイム分析もClojureで試してみようと思います。

<!-- more -->


### アーキテクチャ

Dockerコンテナの監視をどうしようか、[Salt](/2014/09/17/salt-tutorials-monitoring/)や[NewRelic](/2014/08/27/celery-ironmq-newrelic-in-docker-container/)、Mackerelでいろいろと試しましたが、最終的に以下の構成で設計しようと思います。

* [cAdvisor](https://github.com/google/cadvisor)
* [collectd](https://github.com/collectd/collectd)
* [Riemann](https://github.com/aphyr/riemann)
* [InfluxDB](https://github.com/influxdb/influxdb)
* [Grafana](https://github.com/torkelo/grafana)

[cAdvisor](https://github.com/google/cadvisor)はKubernetesやPanamaxで使っていたので知ったのですが、Googleが開発しているコンテナのリソース状況をグラフにしてくれるツールです。まだ実装されていない機能も多いですがadvisorとかalertingとか使えるようになるとうれしいです。

最近Clojureが楽しいので周辺のツールを調べていると[Riemann](https://github.com/aphyr/riemann)のリアルタイム分析システムを知りました。ここから[Riemann + InfluxDB + Grafana](http://www.slideshare.net/nickchappell/pdx-devops-graphite-replacement)というスライドを見つけたので参考にさせていただきます。

[InfluxDB](https://github.com/influxdb/influxdb)はGoでできた時系列データベースです。cAdvisorのバックエンドに使えたり、Riemannからフォワードすることもできます。`Treasure Data`を使っていると時系列データベースの良さに気づいたので、もっと本格的に調べてみたくなりました。

### Monitorama PDX 2014

[Monitorama PDX 2014](http://monitorama.com/)というイベントが5月にありました。先ほどのスライドの最後にリンクが書いてあり、他の動画も[Vimeo](http://vimeo.com/monitorama)にたくさんありました。

* [Monitorama PDX 2014 Grafana workshop](http://vimeo.com/95316672)
* [Monitorama PDX 2014 InfluxDB talk](http://vimeo.com/95311877)
* [Monitorama Boston 2013 Riemann talk](http://vimeo.com/67181466)

来年はできたらポートランドに行って参加したいです。
