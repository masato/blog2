title: 'OpenShift Origin v3 on Kubernetes - Part1: Requirements '
date: 2014-09-11 03:22:56
tags:
 - OpenShiftv3
 - OpenShiftOrigin
 - OpenShift
 - Kubernetes
 - Deis
 - CloudFoundry
 - IDCFクラウド
description: CloudFoundryに押され気味だったのですが、DockerやKubernetesへの対応が速いOpenShiftが最近おもしろそうです。Red Hatが持つ知識と経験のおかげで、Kubernetesは企業向けとして実用可能になったとか。Kubernetesとの統合を試すためにOpenShiftのドキュメントを読み始めたのですが、バージョンの違いがよくわかりません。OpenShift Orignは現在3と4があります。
---

* `Update 2014-09-13`: [OpenShift Origin v3 on Kubernetes - Part2: Ubuntuにインストール](/2014/09/13/openshift-origin-3-on-kubernetes-on-ubuntu/)

CloudFoundryに押され気味だったのですが、DockerやKubernetesへの対応が速いOpenShiftが最近おもしろそうです。
[Red Hatが持つ知識と経験のおかげで、Kubernetesは企業向けとして実用可能になった](http://readwrite.jp/archives/12468)とか。
Kubernetesとの統合を試すためにOpenShiftのドキュメントを読み始めたのですが、バージョンの違いがよくわかりません。`OpenShift Orign`は現在3と4があります。

<!-- more -->

### OpenShift Origin v3と OpenShift Origin V4

08-14に公開された[origin](https://github.com/openshift/origin)のリポジトリは`OpenShift v3`用です。

一方の`OpenShift V4`は、07-20に[Announcing the Release of OpenShift Origin V4](https://www.openshift.com/blogs/announcing-the-release-of-openshift-origin-v4)リリースされています。
リポジトリは以前からある[origin-server](https://github.com/openshift/origin-server)です。

`OpenShift v3`がDockerとKubernetesを基盤にした`third generation`の`v3`なので、`V4`の前の`V3`と明確な違いがある気がするのですが。

さらに、[oo-install User’s Guide](http://openshift.github.io/documentation/oo_install_users_guide.html)を読むと次のように書いてあります。

> OpenShift Origin version 4 is supported on Red Hat Enterprise Linux (RHEL) 6.4 or higher and CentOS 6.4 or higher, 64 bit architecture only. This version is not supported on Fedora, RHEL 7.x or CentOS 7.x.

`V4`はRHEL7.xでは動作しないようです。`6.4 or higher`ですが、`7.x`はサポートされないです。RHEL7からDockerが採用されたのでV4が動作しないのか、ドキュメントが適当な気がします。

### Kubernetesと統合する場合はOpenShift Origin v3を使う

今回の目的はKubernetesと統合して使うことなので、`OpenShift Origin v3`を使います。
従って、以下の[ワンライナー](https://install.openshift.com/)でインストールしません。

``` bash
sh <(curl -s https://install.openshift.com/)
```

### KubernetesのpluginとしてのOpenShift

Kubernetesのプラグインとして、`OpenShift Origin v3`はGoで書かれています。
これだけでは`OpenShift Origin v3`がどんな役割を果たすのかよくわかりません。
`Docker Hub`からイメージをpullするだけなら、普通のKubernetesと変わらないです。

[OpenShift 3.x System Design](https://github.com/openshift/openshift-pep/blob/master/openshift-pep-013-openshift-3.md)を読むと、ユーザーが`git push`するとイメージをビルドして`Docker Registry`にpushしてくれるところが、OpenShiftの提供する`application centric`な役割のようです。

従って、Deisなど他の`PaaS`と同様な位置づけになります。Deisはfleetでスケジュールして、OpenShiftの場合はPodでスケジュールするところが違います。粒度の細かいコンテナのスケジュール管理にはPodが向いていますが、Heroku的な`git push`によるデプロイの概念とPodのグループをどうつながるのか気になります。

### まとめ

一応今日の理解をまとめます。ちゃんと理解できたら書き直します。

* `OpenShift Origin v3` : DockerとKubernetesベース: [origin](https://github.com/openshift/origin)
* `OpenShift Origin V4`: 従来のGearベース: [origin-server](https://github.com/openshift/origin-server)

