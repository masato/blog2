title: 'OpenShift Origin v3 on Kubernetes - Part2: Ubuntuにインストール'
date: 2014-09-13 00:22:29
tags:
 - OpenShiftv3
 - OpenShiftOrigin
 - OpenShift
 - Kubernetes
 - Ubuntu
 - IDCFクラウド
description: Ubuntu14.04のsalt-minionにOpenShift Orign v3をスタンドアロンでインストールします。build-go.shのビルドスクリプトを実行すると、Kubernetesもインストールされます。KubernetesとDockerをgeardとgearの代わりに採用した形でしょうか。RedHatも思い切ってKubernetesとの統合を考えているようです。
---

Ubuntu14.04のsalt-minionに`OpenShift Orign v3`をスタンドアロンでインストールします。`build-go.sh`のビルドスクリプトを実行すると、Kubernetesもインストールされます。
KubernetesとDockerをgeardとgearの代わりに採用した形でしょうか。RedHatも思い切ってKubernetesとの統合を考えているようです。

<!-- more -->

### Ubuntu14.04にインスト-ル

[Saltで準備](/2014/09/12/salt-tutorials-openshift-3-prepare/)した、Ubuntu14.04にインスト-ルします。

``` bash
$ cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04.1 LTS"
```

### OpenShift Orign v3 のインストール準備

[Contributing to OpenShift 3](https://github.com/openshift/origin/blob/master/CONTRIBUTING.adoc)の手順に従ってインストールしていきます。まず前提条件になているGoとDokcerのインストールを確認します。

Goインストールの確認をします。

``` bash
$ go env GOROOT
/usr/lib/go
$ go env GOPATH
/home/mshimizu/go
```

Dockerインストールの確認をします。

``` bash
$ sudo docker version
Client version: 1.2.0
Client API version: 1.14
Go version (client): go1.3.1
Git commit (client): fa7b24f
OS/Arch (client): linux/amd64
Server version: 1.2.0
Server API version: 1.14
Go version (server): go1.3.1
Git commit (server): fa7b24f
```

### OpenShift Origin v3 のビルド 
 
`go get`でGoパッケージをインストールします。

``` bash
$ go get github.com/openshift/origin
```

パッケージのディレクトリに移動して、ビルドをします。

``` bash
$ cd $GOPATH/src/github.com/openshift/origin
$ hack/build-go.sh
```

### OpenShift Origin v3 の起動 

`OpenShift Origin v3`を起動します。

``` bash
$ _output/go/bin/openshift start
...
I0911 12:32:09.144353 05618 log.go:134] GET /healthz: (4.996us) 200
I0911 12:32:09.144530 05618 log.go:134] GET /api/v1beta1/minions: (403.509us) 200
```

`openshift kube create pods`コマンドを使いKubernetesデプロイをします。

``` bash
$ cd $GOPATH/src/github.com/openshift/origin
$ sudo _output/go/bin/openshift kube create pods -c examples/hello-openshift/hello-pod.json
ID                  Image(s)                    Host                Labels                 Status
----------          ----------                  ----------          ----------             ----------
hello-openshift     openshift/hello-openshift   /                   name=hello-openshift   Waiting
```

KubernetesのPodsを一覧します。

``` bash
$ sudo _output/go/bin/openshift kube list pods
ID                  Image(s)                    Host                Labels                 Status
----------          ----------                  ----------          ----------             ----------
hello-openshift     openshift/hello-openshift   127.0.0.1/          name=hello-openshift   Running
```

### 確認

curlでPodのコンテナの動作を確認します。

``` bash
$ curl -s http://10.1.3.67:6061
Hello OpenShift!
```

### まとめ

`OpenShift Origin V4`はRHEL6.4以上(7.xは不可)にしかインストールできませんでしたが、`OpenShift Origin v3`はKubernetesをベースにしているので、RedHat以外のLinuxにもインストールできます。

これまでの`OpenShift Origin V4`はどうなっていくのか気になりますが、Kubernetesへのシフトはよいことだと思います。

