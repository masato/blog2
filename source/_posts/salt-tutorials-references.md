title: 'Salt チュートリアル - Part1: リファレンス'
date: 2014-09-09 00:12:27
tags:
 - Salt
 - Kubernetes
 - cloud-config
description: CoreOSクラスタにcloud-configを使いKubernetesをインストールしましたが、通常GCEやAzureに構築する場合はSalt使いbootstrapします。SaltStack’s Big Role in Google Kubernetes and Why Immutable Infrastructure Makes The Cloud a Giant Computerという記事に、CoreOSのアプローチが既存の構成管理ツールに与える影響について書いてあります。cloud-configを書いてCoreOSクラスタを構築していると、既存のCMツールの問題も見えてきます。まだCoreOSは主流でないのでSaltの役割は大きいです。Saltを習得するための教材をいくつか集めました。Saltはオンラインドキュメントの文量はかなりあるのですが全体を把握するのが困難です。
---

* `Update 2014-09-10`: [Salt チュートリアル - Part2: Reactor System](/2014/09/10/salt-tutorials-reactor-sytem/)
* `Update 2014-09-12`: [Salt チュートリアル - Part3: OpenShift Origin 3の準備](/2014/09/12/salt-tutorials-openshift-3-prepare/)


[CoreOSクラスタにcloud-configを使いKubernetesをインストール](/2014/09/04/kubernetes-in-coreos-on-idcf-with-rudder/)しましたが、通常GCEやAzureに構築する場合はSalt使いbootstrapします。

[SaltStack’s “Big” Role in Google Kubernetes and Why Immutable Infrastructure Makes The Cloud a Giant Computer](http://thenewstack.io/saltstacks-big-role-in-google-kubernetes-and-why-immutable-infrastructure-makes-the-cloud-a-giant-computer/)という記事に、CoreOSのアプローチが既存の構成管理ツールに与える影響について書いてあります。cloud-configを書いてCoreOSクラスタを構築していると、既存のCMツールの問題も見えてきます。

まだCoreOSは主流でないのでSaltの役割は大きいです。Saltを習得するための教材をいくつか集めました。Saltはオンラインドキュメントの文量はかなりあるのですが全体を把握するのが困難です。

<!-- more -->

### Kubernetesのインストールスクリプト

* [インストールスクリプト](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/cluster)

GitHubにIaaSごとのシェルスクリプトを読むと、Saltとてもよい勉強になります。`Reactor System`も使っています。

### Infrastructure Management with SaltStack

* [Infrastructure Management with SaltStack: Part 1 – The Setup](http://vbyron.com/blog/infrastructure-management-saltstack-part-1-getting-started/)
* [Infrastructure Management with SaltStack: Part 2 – Grains, States, and Pillar](http://vbyron.com/blog/infrastructure-management-saltstack-part-2-grains-states-pillar/)
* [Infrastructure Management with SaltStack: Part 3 – Reactor and Events](http://vbyron.com/blog/infrastructure-management-saltstack-part-3-reactor-events/)
 
3部構成でわかりやすいチュートリアルです。Salt独特の概念も丁寧に解説してくれます。
特に`Reactor System`はここでようやく理解できました。

### InfoQ

* [SaltStack for Flexible and Scalable Configuration Management](http://www.infoq.com/articles/saltstack-configuration-management)

>Infrastructure as data, not code

Saltは手続き型のようにも書けますが、できるだけ宣言型で書いていきたいです。プログラミングも関数型言語の方が好みになってきたので。

### Rackspace Developer Blog

* [Marconi and Salt, part 1](https://developer.rackspace.com/blog/marconi-and-salt/)
* [Marconi and Salt: Part 2](https://developer.rackspace.com/blog/marconi-and-salt-part-2/)
 
SLSファイルの書き方がわかりやすいです。

### Salt Essentials

* [Salt Essentials](http://shop.oreilly.com/product/0636920033240.do)

O'Reillyから`Early Release`で出版されています。[Back to (Tech) School Sale](http://shop.oreilly.com/category/deals/b2s-special.do?code=B2S4)中のため半額で買えました。
はじめて使うツールなど、概念から学習する場合は書籍が最適です。

