title: "Kubernetes in CoreOS with Rudder on IDCFクラウド - Part2: クラスタ再考"
date: 2014-08-31 21:10:24
tags:
 - Docker管理
 - Kubernetes
 - CoreOS
 - Salt
 - SDN
 - IDCFクラウド
description: グーグルが自社データセンターをオープンソース化した方法とその理由という記事を読みました。Docker管理の側面から見ていたのですが「Dockerアプリケーションの運用も可能になる」が正しく、KubernetesからGoogleのデータセンターとアプリ開発、管理の秘密を垣間見る価値の大きさをあらためて感じます。GoogleのCraig McLuckieさんが、以下のように言っています。我々が社内で得ているのと同じ恩恵を顧客にも得てほしい、と思っています。Kubernetesを使って、我々が社内で使うのと似たようなポータブルで無駄の無いオープンソース・システムで、アプリケーションを作成、管理に役立てて欲しいスタンドアロンのCoreOSにKubernetesをマニュアルインストールできた程度なのですが先に進みません。Kubernetesの中で、SaltとCoreOSとMesosとの関係が混乱してきたので自分なりに整理しようと思います。Googleのオープンソースへの関わり方に興味が尽きません。
---

* `Update 2014-10-02`: [Kubernetes on CoreOS with Fleet and flannel on IDCFクラウド - Part7: flannel版GuestBook](/2014/10/02/kubernetes-fleet-flannel-guestbook/)
* `Update 2014-09-27`: [Kubernetes on CoreOS with Fleet and flannel on IDCFクラウド - Part6: Rudder renamed to flannel](/2014/09/27/kubernetes-fleet-rudder-renamed-to-flannel/)
* `Update 2014-09-22`: [Kubernetes on CoreOS with Fleet and Rudder on IDCFクラウド - Part5: FleetとRudderで再作成する](/2014/09/22/kubernetes-fleet-rudder-on-idcf/)
* `Update 2014-09-05`: [Kubernetes in CoreOS with Rudder on IDCFクラウド - Part4: GuestBook example](/2014/09/05/kubernetes-in-coreos-on-idcf-guestbook-example/)
* `Update 2014-09-04`: [Kubernetes in CoreOS with Rudder on IDCFクラウド - Part3: RudderでSDN風なこと](/2014/09/04/kubernetes-in-coreos-on-idcf-with-rudder/)


[グーグルが自社データセンターをオープンソース化した方法とその理由](http://readwrite.jp/archives/12468)という記事を読みました。Docker管理の側面から見ていたのですが「Dockerアプリケーションの運用も可能になる」が正しく、KubernetesからGoogleのデータセンターとアプリ開発、管理の秘密を垣間見る価値の大きさをあらためて感じます。

GoogleのCraig McLuckieさんが、以下のように言っています。

>我々が社内で得ているのと同じ恩恵を顧客にも得てほしい、と思っています。Kubernetesを使って、我々が社内で使うのと似たようなポータブルで無駄の無いオープンソース・システムで、アプリケーションを作成、管理に役立てて欲しい

スタンドアロンのCoreOSにKubernetesをマニュアルインストールできた程度なのですが先に進みません。
Kubernetesの中で、SaltとCoreOSとMesosとの関係が混乱してきたので自分なりに整理しようと思います。

Googleのオープンソースへの関わり方に興味が尽きません。

<!-- more -->

### Kubernetes configured using Salt

Saltの役割は予めIaaSに作成したインスタンスに対して、Kubernetesの構成管理を行うことです。

[クラスタ構築のシェルスクリプト](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/cluster)には、IaaSごとのCLI実行とsalt-bootstrapの設定、Saltによる構成管理がまとめてあります。

* IaaSのCLIでインスタンスを作成
* インスタンス起動時に`salt-bootstrap`が実行されSalt master/minionのインストール
* SaltでKubernetesクラスタの構成管理

たとえば、[Azureのデプロイスクリプト](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/cluster/azure/util.sh)の場合、インスタンス作成はsalt-cloudではなく、シェルスクリプトから`azure vm create`コマンドの`-d`オプションを使い`salt-bootstrap`をcustom-dataとして渡しています。


### Kubernetes on CoreOS

CoreOS上にKubernetesクラスタの構成管理とスケジュール管理を行います。
今のところkubeletなどの管理サービスは直接OSにインストールされるので、管理サービスもコンテナとしてfleetで管理できるメリットもあります。

IaaS毎にKubernetesクラスタを構築するまでの方法は異なるので、IaaS上にCoreOSクラスタにネットワークも含めて抽象化して、Kubernetesを動かすのは合理的だと思います。

[Rudder](https://github.com/coreos/rudder)というツールで、CoreOSのKubernetesへの貢献もでてきましたが、フットワークの軽さと技術力の高さが見えます。

systemdにfleetの要素が追加されますが、ここはがんばって理解しようと思います。[master.yml](https://github.com/kelseyhightower/kubernetes-coreos/blob/master/configs/master.yml)くらいのcloud-configは自分でも書けるようにならないと。

Podとコンテナのスケジュール管理はfleetでなくKubernetesが行います。fleetは低レベルなため粒度を細かくしたコンテナの管理を直接表現するのは難しい感じがするので、Kubernetesに任せるの良いと思います。スケジュール管理以外にもプロキシーにVulcandを使うよりよい気もします。

fleetの抽象化のため、CoreOS上にPanamaxやDeisでコンテナ管理をするのも良い案です。

### Kubernetes on Meeos

MesosphereにはDatacenter-as-a-ComputerというGoogleに影響を受けた壮大な考えがあります。
データセンター全体を大きなリソースプールとして、KubernetesもSparkやHadoopなどと同様にリソースを共有して使いたい場合に、Mesosは最適です。

Mesos自体はクラスタ全体の最適なリソース管理を行うため、これまでDockerのスケジュールはMarathonが行っていました。今回の提携により、Dockerの場合はMarathonがKubernetesに置き換わるのでしょうか。

Mesosのアーキテクチャは理想的ですが、現実としてSparkとHadoopとDockerを共存させるほどの巨大なプールが必要になっていないので様子見します。

### まとめ

今後の学習方針をまとめました。

1. Kubernetesインスタンスは、IaaSが提供するCLIを活用してデプロイする、salt-cloudは使わない
2. インスタンスはCoreOSを使い、Kubernetes構成管理はcloud-configを使う
3. 1と2はシェルスクリプトでまとめ、Kubernetes構築を自動化する
4. コンテナのスケジュール管理はKubernetesを使う
5. CoreOSでKubernetesの構築ができたら、その他LinuxOSへSaltを使いKubernetesの構成管理をする(3,4を行う)

いずれにせよ、Kubernetesから目が離せないです。
