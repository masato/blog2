title: 'SaltとDockerでクラウドプロビジョニング'
date: 2014-05-19 01:22:46
tags:
 - Salt
 - Docker
 - GCE
 - DigitalOcean
 - CoreOS
 - ProjectAtomic
description: GCEにプロビジョニングした方法と同じように、Saltを使うとDigitalOceanやRackspaceにもプロビジョニングできます。さらにDockerと組み合わせてイメージのpushと同時に、複数のクラウドへインスタンス作成とDockerコンテナの起動までできるようになります。
---

* `Update 2014-09-06`: [Salt with Docker - Part1: Dockerインストール](/2014/09/06/salt-idcf-docker-states/)
* `Update 2014-09-07`: [Salt with Docker - Part2: WordPressインストール](/2014/09/07/salt-idcf-docker-wordpress/)


GCEに[プロビジョニング](/2014/05/15/gce-salt-cloud/)した方法と同じように、Saltを使うとDigitalOceanやRackspaceにもプロビジョニングできます。
さらにDockerと組み合わせてイメージのpushと同時に、複数のクラウドへインスタンス作成とDockerコンテナの起動まで
できるようになります。

<!-- more -->

[Automating application deployments across clouds with Salt and Docker](http://thomason.io/automating-application-deployments-across-clouds-with-salt-and-docker/)や[Partial Continuous Deployment With Docker and SaltStack](http://bitjudo.com/blog/2014/05/13/partial-continuous-deployment-with-docker-and-saltstack/)に、DitigalOceanをつかったプロビジョニングとデプロイ方法が書いてありました。

[Salt Clouds](http://docs.saltstack.com/ref/clouds/all/index.html)と[Salt States](http://docs.saltstack.com/en/latest/ref/states/all/salt.states.dockerio.html)と、[Docker Remote API](http://docs.docker.io/reference/api/docker_remote_api/)を組み合わせると、`Immutable Infrastructure`環境を始めることができそうです。

GCEやDigitalOceanはインスタンスの起動が速いため、CoreOSや[Project Atomic](http://www.projectatomic.io/)などの軽量OSをDokcerホストにすると、結構いけるのではないかと思いました。

これからしばらくは、Salt + Docker + GCE + CoreOS or `Project Atomic`を中心にして、今後のクラウドインフラを考えてみます。
