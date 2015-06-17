title: "Spring BootをPivotal Web Servicesにデプロイする - Part1: ClearDBを使う"
date: 2014-12-08 20:17:02
tags:
 - SpringBoot
 - SpringCloud
 - ClearDB
 - ElephantSQL
 - PivotalWebServices
 - PivotalCF
 - CloudFoundry
description: もうすぐPivotal Web Servicesの60日間無料トライアルが期限切れになるので、急いでアプリを作らないといけないです。Hello Worldからすこし成長して、RDBを使ったプログラムをデプロイしようと思います。通常PaaSでRDBを使う場合は、Amazon RDSのようなDBaaSを使う必要があります。IronMQとの連携の際に、Amazon RDSの代替サービスとしてPostgreSQLのElephantSQLを使いました。今度はMySQLのDBaaSであるClearDBを試してみます。
---

もうすぐ[Pivotal Web Services](https://run.pivotal.io/)の60日間無料トライアルが期限切れになるので、急いでアプリを作らないといけないです。Hello Worldからすこし成長して、RDBを使ったプログラムをデプロイしようと思います。通常PaaSでRDBを使う場合は、Amazon RDSのようなDBaaSを使う必要があります。[IronMQとの連携](/2014/09/02/celery-ironmq-elephantsql/)の際に、Amazon RDSの代替サービスとしてPostgreSQLの[ElephantSQL](http://www.elephantsql.com/)を使いました。今度はMySQLのDBaaSである[ClearDB](https://www.cleardb.com/)を試してみます。

<!-- more -->

### PaaS向けの共有無料プラン

ClearDBはいろいろなPaaSのアドオンとして使える、開発用の[無料プラン](https://www.cleardb.com/pricing/multitenant.view)が用意されています。共有サービスでストレージも5MBと少ないですが、動作確認には十分です。どのPaaSもプラン名はおもしろいです。

* Pivotal Web Services: Spark DB / 5MB
* AppFog: Aluminium DB　/ 5MB
* Heroku: Ignite DB / 5MB
* MicrosoftAzure: Mercury / 20MB

今回はPivotal Web Servicesにログインしてから、無料プランを使ってみます。

### 通常の専有クラスタープラン

PaaS以外から使う場合は[専有プラン](https://www.cleardb.com/pricing/missioncritical.view)になりますが、最小のSmall DBでも$399.99/moなので、個人で使うには高めの設定です。30日間の無料期間があるのでこちらも試してみようと思います。


### Pivotal Web Services用のClearDB

Merketplace画面から、CleaDBを選択します。SERVICE PLANSからfreeのSpark DBを選び、"Select this plan"ボタンを押すと、ClearDBのMy Dashboardに移動します。
My Databaseに一覧されているDatabase nameをクリックすると、データベースのページが表示されるので、"Endpoint Inforation"タブからDB接続情報を確認します。

ローカルからもmysqlコマンドで接続できました。MySQLのバージョンは5.5.40です。比較的最新のバージョンを使えるようです。

``` bash
$ mysql -h xxx.cleardb.net -uxxx -pxxx xxx
...
Server version: 5.5.40-log MySQL Community Server (GPL)
```

PivotalCFのClearDBの[ドキュメント](http://docs.run.pivotal.io/marketplace/services/cleardb.html)を読むと、環境変数`VCAP_SERVICES`のフォーマットは以下のようなJSONになっています。このJSONをパースすれば接続情報をプログラムから取得できますが、どうもスマートな方法ではありません。

``` json VCAP_SERVICES 
{
  cleardb: [
    {
      name: "cleardb-ebf69",
      label: "cleardb",
      plan: "spark",
      credentials: {
        jdbcUrl: "jdbc:mysql://bb76fcb43f807b:6776566b@us-cdbr-aws-east-105.cleardb.net:3306/ad_033c9629217c759",
        uri: "mysql://bb76fcb43f807b:6776566b@us-cdbr-aws-east-105.cleardb.net:3306/ad_033c9629217c759?reconnect=true",
        name: "ad_033c9629217c759",
        hostname: "us-cdbr-aws-east-105.cleardb.net",
        host: "us-cdbr-aws-east-105.cleardb.net",
        port: "3306",
        user: "bb76fcb43f807b",
        username: "bb76fcb43f807b",
        password: "6776566b"
      }
    }
  ]
}
```

### Spring Cloudの@Profile("cloud")を使う

ここでSpring Cloudの出番のようです。[Configure Service Connections for Spring](http://docs.cloudfoundry.org/buildpacks/java/spring-service-bindings.html)を読むと何となくうまく行きそうな感じがします。
