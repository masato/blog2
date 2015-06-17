title: 'IDCFクラウドにCoreOSクラスタを構築する - Part2: CoreOSのディスカバリ'
date: 2014-07-10 22:03:10
tags:
 - IDCFクラウド
 - CoreOS
 - ServiceDiscovery
 - etcd
 - etcdio
description: IDCFクラウドでCoreOSをディスクからインストールするからしばらく経ちました。TechClunchにMicrosoft、Red Hat、IBM等がGoogleのDockerコンテナ管理ツール、Kubernetesサポートで団結いう記事を読んだら、Salt,CoreOS,Mesosの3つは速くマスターしないといけないです。IDCFクラウドだと手順が煩雑なのですが、CoreOSクラスタを再構築のためdiscovery.etcd.ioの操作を確認します。

---

* `Update 2014-09-22`: [Kubernetes on CoreOS with Fleet and Rudder on IDCFクラウド - Part5: FleetとRudderで再作成する](/2014/09/22/kubernetes-fleet-rudder-on-idcf/)
* `Update 2014-07-19`: [IDCFクラウドにCoreOSクラスタを構築する - Part5: 再セットアップ](/2014/07/19/idcf-coreos-cluster-re-setting-up/)
* `Update 2014-07-17`: [IDCFクラウドにCoreOSクラスタを構築する - Part4: 367.1.0のissue](/2014/07/17/idcf-coreos-cluster-36710/)
* `Update 2014-07-15`: [IDCFクラウドにCoreOSクラスタを構築する - Part3: クラスタをセットアップ](/2014/07/15/idcf-coreos-cluster-setting-up)

[IDCFクラウドでCoreOSをディスクからインストールする](/2014/06/03/idcf-coreos-install-disk/)からしばらく経ちました。
TechClunchに[Microsoft、Red Hat、IBM等がGoogleのDockerコンテナ管理ツール、Kubernetesサポートで団結](http://jp.techcrunch.com/2014/07/11/20140710google-microsoft-ibm-and-others-collaborate-to-make-managing-docker-containers-easier/)という記事を読んだら、Salt,CoreOS,Mesosの3つは速くマスターしないといけないです。

IDCFクラウドだと手順が煩雑なのですが、CoreOSクラスタを再構築のためdiscovery.etcd.ioの操作を確認します。

<!-- more -->

### discovery.etcd.io からTOKENの取得
discovery.etcd.ioからTOKENを取得します。

``` bash
$ curl https://discovery.etcd.io/new
https://discovery.etcd.io/a876553c49f52f4a3a5a91b5bbdca88c
```

discovery.etcd.io取得したTOKENをdiscoveryに使用します。
ローカルでCoreOSクラスタを構築するので、各nodeはプライベートIPアドレスで見つかるようにします。

``` yaml ~/coreos_apps/cloud-config.yml
#cloud-config

ssh_authorized_keys:
  - ssh-rsa xxxx

coreos:
  etcd:
      discovery: https://discovery.etcd.io/1461ea8c611984341b01f611ff5940e1
      addr: $private_ipv4:4001
      peer-addr: $private_ipv4:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
```

### discovery.etcd.io のTOKENからノードの削除

CoreOSをディスクにインストールする場合、TOKENがハードコードしてしまいます。
根本的に解決はしないのですが、新しいクラスタを作る場合にディスカバリをリセットしたいです。

[Howto Reset etcd discovery](https://gist.github.com/skorfmann/10243181)にURLのリセット方法がありました。

``` bash
$ curl -XDELETE https://discovery.etcd.io/TOKEN/_state
```

削除したいノードが残っている状態です。

``` bash
$ curl https://discovery.etcd.io/1461ea8c611984341b01f611ff5940e1
{"action":"get","node":{"key":"/_etcd/registry/1461ea8c611984341b01f611ff5940e1","dir":true,"nodes":[{"key":"/_etcd/registry/1461ea8c611984341b01f611ff5940e1/2a631c0e82604e93bc727887a01fc3b8","value":"http://10.1.0.73:7001","expiration":"2014-07-19T00:16:14.12997006Z","ttl":604575,"modifiedIndex":49748734,"createdIndex":49748734}],"modifiedIndex":49375252,"createdIndex":49375252}}
```

`_state`キーを確認します。

``` bash
$ curl https://discovery.etcd.io/1461ea8c611984341b01f611ff5940e1/_state
{"action":"get","node":{"key":"/_etcd/registry/1461ea8c611984341b01f611ff5940e1/_state","value":"started","modifiedIndex":49748735,"createdIndex":49748735}}
```

`_state`キーを削除します。

``` bash
$ curl -XDELETE https://discovery.etcd.io/1461ea8c611984341b01f611ff5940e1/_state
{"action":"delete","node":{"key":"/_etcd/registry/1461ea8c611984341b01f611ff5940e1/_state","modifiedIndex":49751193,"createdIndex":49748735}}
```

削除できているか`_state`キーを確認します。

``` bash
$ curl https://discovery.etcd.io/1461ea8c611984341b01f611ff5940e1/_state
{"errorCode":100,"message":"Key not found","cause":"/_etcd/registry/1461ea8c611984341b01f611ff5940e1/_state","index":49751488}
```

`_state`キーが削除できたので、ディスカバリの失敗は発生しなくなりますが、ノードの登録はexpirationするまで残りそうです。

``` bash
$ curl https://discovery.etcd.io/1461ea8c611984341b01f611ff5940e1
{"action":"get","node":{"key":"/_etcd/registry/1461ea8c611984341b01f611ff5940e1","dir":true,"nodes":[{"key":"/_etcd/registry/1461ea8c611984341b01f611ff5940e1/2a631c0e82604e93bc727887a01fc3b8","value":"http://10.1.0.73:7001","expiration":"2014-07-19T00:16:14.12997006Z","ttl":604440,"modifiedIndex":49748734,"createdIndex":49748734}],"modifiedIndex":49375252,"createdIndex":49375252}}
```

### discovery.etcd.io のTOKENからノードの削除

[discovery.etcd.io](https://github.com/coreos/discovery.etcd.io)がGitHubにあるのでソースコードを読んでみます。[http.go](https://github.com/coreos/discovery.etcd.io/blob/master/http/http.go)にGorillaのハンドラ登録がありました。
`/{token:[a-f0-9]{32}}/{machine}`を渡してDELTEできそうです。

``` go http.go
func init() {
	r := mux.NewRouter()

	r.HandleFunc("/", handlers.HomeHandler)
	r.HandleFunc("/new", handlers.NewTokenHandler)
	r.HandleFunc("/health", handlers.HealthHandler)

	// Only allow exact tokens with GETs and PUTs
	r.HandleFunc("/{token:[a-f0-9]{32}}",
				handlers.TokenHandler).
		Methods("GET", "PUT")
	r.HandleFunc("/{token:[a-f0-9]{32}}/",
				handlers.TokenHandler).
		Methods("GET", "PUT")
	r.HandleFunc("/{token:[a-f0-9]{32}}/{machine}",
				handlers.TokenHandler).
		Methods("GET", "PUT", "DELETE")

	logH := gorillaHandlers.LoggingHandler(os.Stdout, r)

	http.Handle("/", logH)
}
```

ノードを指定して削除します。

``` bash
$ curl -XDELETE https://discovery.etcd.io/1461ea8c611984341b01f611ff5940e1/2a631c0e82604e93bc727887a01fc3b8
{"action":"delete","node":{"key":"/_etcd/registry/1461ea8c611984341b01f611ff5940e1/2a631c0e82604e93bc727887a01fc3b8","modifiedIndex":49755257,"createdIndex":49748734}}
```

確認してみます。ノードのエントリが削除できました。

``` bash
$ curl https://discovery.etcd.io/1461ea8c611984341b01f611ff5940e1
{"action":"get","node":{"key":"/_etcd/registry/1461ea8c611984341b01f611ff5940e1","dir":true,"modifiedIndex":49375252,"createdIndex":49375252}}
```

### まとめ

これが正しい手順なのかよくわかりませんが、試行錯誤の段階なのでこのまま進んで、IDCFクラウド上にCoreOSのクラスタを構築してみます。


