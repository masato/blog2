title: "Micro Services with Docker or OSv - Part1: DockerとOSvとSpring Cloudを考える"
date: 2014-10-03 01:25:14
tags:
 - MicroServices
 - MicroOS
 - Docker
 - OSv
 - NetflixOSS
 - SpringCloud
 - SpringBoot
description: Dockerコンテナを細かく機能で分割するデザインをしていると、以前Dropwizardやvert.xで実装を試していたMicro Servicesと似ている気がしてきました。Chris RichardsonのMicroervices Decomposing Applications for Deployability and Scalabilityのスライドは懐かしい名前です。
---

* `Update 2014-10-04`: [Docker開発環境をつくる - Emacs24とEclimからEclipseに接続するJava開発環境](/2014/10/04/docker-devenv-emacs24-eclim-java/)

Dockerコンテナを細かく機能で分割するデザインをしていると、以前[Dropwizard](https://github.com/codahale/dropwizard)や[vert.x](http://vertx.io/)で実装を試していた[Micro Services](http://martinfowler.com/articles/microservices.html)と似ている気がしてきました。

Chris Richardsonの[Microervices: Decomposing Applications for Deployability and Scalability](http://www.slideshare.net/chris.e.richardson/microservices-decomposing-applications-for-deployability-and-scalability-jax)は懐かしい名前です。

<!-- more -->

### OSv

[OSv](http://osv.io/)というクラウドのために設計されたハイパバイザ-上で動作する専用OSがあります。
`Library OS`とも言えるJVMと機能を削りライブラリに近いOSを、ホストOSのハイパバイザー上で直接動かすというかなり思い切った実装です。

OSvは`Zero OS Management`が可能で、クラウド上のゲストOSの構成管理や設定が不要になるのは、CoreOSやProjectAtomicとコンセプトは似ています。
OSvはステートレスなライブラリに近く、DockerコンテナでなくJVMを動かすことに特化しているので、より軽量でOSの存在を意識させません。
そもそもJVMはチューニングの時に存在を意識するくらいなので、プログラマがアプリだけに集中できます。PaaSとかDevOpsとか急速に過去のものに感じてしまいます。

>The bloated legacy UNIX configurations are gone. We’re stateless! No need for administration, template management, configuration and tuning

JVM言語のアプリを仮想マシンで動かすと、ゲストOSの存在が無駄な気がしていて、VMwareがJVMを統合したハイパーバイザーを出してくれないかなと思っていたくらいです。

One-JARのプログラムを動かすためのDockerfileはJDKをインストールしてJARをADDするだけなので非常に簡単です。
OSvでもDockerfileに似た概念で記述できる、Goで書かれた[Capstan](http://osv.io/capstan/)というOSv上のVM用のビルドツールがあります。
JVM言語とネイティブアプリですべてシステムを構成できるなら、Dockerでなくてもいいかもと思い始めました。

OSvは1VM=1プロセス=1アプリなのですが、1アプリの粒度をどう設計するのか、複数アプリのディスカバリとオーケストレーションはどうするのかといった問題はDockerコンテナと同様の課題になります。

### Spring BootとSpring Cloud 

Spirng Frameworkはいまどうなったのか気になってしらべていると、Spring BootやSpring Cloudと言ったフレームワークが出ていました。
Spring Bootは軽量なMVCフレームワークとして単独でも使えます。

Spring CloudはMicro ServicesのJVMをDockerコンテナと比較した場合にfleetやKubernetesに相当する、JVMアプリ管理ツールといった感じです。

[Features](http://projects.spring.io/spring-cloud/spring-cloud.html#_features)や、[Cloud Native NetflixOSS Services on Docker](http://www.slideshare.net/dotCloud/cloud-native-netflixoss-services-on-docker)のスライドを読むと、Kubernetes+fleetで実現できることとほとんど同じですがLinuxOSとコンテナの組み合わせよりも、JVMのアプリだけに集中できるため、よりプログラマブルにデザインと開発ができそうです。

* Distributed/versioned configuration
* Service registration and discovery
* Routing
* Service-to-service calls
* Load balancing
* Circuit Breakers
* Global locks
* Leadership election and cluster state
* Distributed messaging

### まとめ

OSvとSpring Cloud、DockerとNetflixOSSとSpring Cloudの組み合わせは、Micro Servicesを継続的に開発、運用していく上で重要なフレームワークになりそうです。本番環境使えそうなSpring BootでWebアプリを作るところから初めて、OSvも実際に動かしてみようと思います。
