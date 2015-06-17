title: "Spring BootをBluemixにデプロイする - Part1: Spring Cloudのauto-reconfigurationが動かない"
date: 2014-12-13 13:18:32
tags:
 - Bluemix
 - CloudFoundry
 - PivotalWebServices
 - PivotalCF
 - SpringBoot
 - SpringCloud
descriptin: ITのメルマガでBluemix無料トライアルスタートキャンペーンの広告をみつけました。30日間無料で使えて先着400名にiTunesギフト券2500円分があたります。クレジットカードの入力も不要なので、さっそく個人アカウントを作成して、先日Pivotal Web ServicesにデプロイしたSpring Bootのjarを、変更をすることなくBluemixにもデプロイできるか試してみます。
---

@ITのメルマガで[Bluemix無料トライアルスタートキャンペーン](http://www.atmarkit.co.jp/ait/articles/1411/27/news013.html)の広告をみつけました。30日間無料で使えて先着400名にiTunesギフト券2500円分があたります。クレジットカードの入力も不要なので、さっそく個人アカウントを作成して、先日[Pivotal Web Servicesにデプロイ](/2014/12/10/spring-boot-pivotal-web-services-cleardb-deploy/)したSpring Bootのjarを、変更をすることなくBluemixにもデプロイできるか試してみます。

<!-- more -->

### Liberty for Java

Spring BootのLiberty for Javaアプリとして、Liberty Buildpackを使います。ドキュメントは[Creating apps with Liberty for Java](https://www.ng.bluemix.net/docs/#starters/liberty/index.html#liberty)にあります。

### Boilerplates

Bluemixにログインしてカタログのページを見ると、Pivotal Web Servicesと異なりBoilerplatesというカテゴリがあります。Boilerplatesは言語のRutimesとRDBなどのサービスがセットで予め構成されているので、やりたいことがBoilerplatesにあれば、まとめて構築できます。

### SQL Database

今回はBoilerplatesを使わず、RDBは個別のサービスを選択します。Bluemixの場合のRDBはSQL DatabaseというDB2のインスタンスを使えます。

### cloudfoundry/cli

CloudFoundryのcfコマンドツールは、[Pivotal Web Services用](/2014/12/10/spring-boot-pivotal-web-services-cleardb-deploy/)にインストールしたバイナリをそのまま使います。エンドポイントをクリアするために一度logoutします。

``` bash
$ cf --version
cf version 6.7.0-c38c991-2014-11-12T01:50:11+00:00
$ cf logout
OK
```

### Bluemixにlogin

Dashboard画面右上のRegionを確認すると`US South`になっています。ドキュメントページの[Bluemix concepts](https://www.ng.bluemix.net/docs/#overview/overview.html#ov_intro)対応表ではエンドポイントは以下になります。

``` bash
$ cf api https://api.ng.bluemix.net
Setting api endpoint to https://api.ng.bluemix.net...
OK


API endpoint:   https://api.ng.bluemix.net (API version: 2.14.0)
Not logged in. Use 'cf login' to log in.
```

最初にcfコマンドから、Bluemixにloginします。

``` bash
$ cf login -a https://api.ng.bluemix.net
```


### Bluemixにpush

プロジェクトにあるgradlewを使い、JARファイルをビルドします。

``` bash
$ git clone https://github.com/masato/spring-cloud-sample
$ cd spring-cloud-sample
$ ./gradlew clean build
$ ls build/libs/
helloconfig-0.0.1-SNAPSHOT.jar  build/libs/helloconfig-0.0.1-SNAPSHOT.jar.original
```

JARファイルをBluemixにデプロイします。JavaのBuildpackのバージョンは1.7.1です。このアプリはJava8でビルドしているので、デプロイに失敗しているようです。downのまま先に進みません。Ctl-Cでプロセスを終了させます。

``` bash
$ cf push masato-hello -p build/libs/helloconfig-0.0.1-SNAPSHOT.jar
...
-----> Downloaded app package (25M)
-----> Liberty Buildpack Version: v1.9-20141202-0947
-----> Retrieving IBM 1.7.1 JRE (ibm-java-jre-7.1-1.0-pxa6470_27sr2ifx-20141115_01-sfj.tgz) ... (0.0s)
         Expanding JRE to .java ... (0.7s)
-----> Retrieving App Management Agent 1.0.0_master (com.ibm.ws.cloudoe.app-mgmt-liberty.zip) ... (0.0s)
         Expanding App Management to .app-management (0.0s)
-----> Downloading Auto Reconfiguration 1.5.0_RELEASE from https://download.run.pivotal.io/auto-reconfiguration/auto-reconfiguration-1.5.0_RELEASE.jar (0.2s)
-----> Liberty buildpack is done creating the droplet
-----> Uploading droplet (77M)
...
0 of 1 instances running, 1 down
0 of 1 instances running, 1 down
```

### Java7でビルドする

build.gradleを編集して、Java7を指定します。

``` groovy ~/spring-cloud-sample/build.gradle
//sourceCompatibility = 1.8
//targetCompatibility = 1.8

sourceCompatibility = 1.7
targetCompatibility = 1.7
```

ビルドをし直して再度pushすると、デプロイに成功しました。

``` bash
$ ./gradlew clean build
$ cf push masato-hello -p build/libs/helloconfig-0.0.1-SNAPSHOT.jar
```

### SQL Databaseサービスインスタンスを作成

MarketplaceからSQL Databaseサービスを探します。

``` bash
$ cf marketplace | grep sql
cf marketplace | grep sql
elephantsql                turtle                                                                           PostgreSQL as a Service
mysql                      100                                                                              MySQL database
postgresql                 100                                                                              PostgreSQL database
sqldb                      sqldb_small, sqldb_free                                                          SQL Database adds an on-demand relational database to your application. Powered by DB2, it provides a managed database service to handle web and transactional workloads.
```

SQL Databaseサービスのインスタンスを作成します。

``` bash
$ cf create-service sqldb sqldb_small mydb
```

### SQL Databaseサービスのバインド

SQL Databaseサービスをアプリケーションとバインドして、アプリケーションをリスタートします。

``` bash
$ cf bind-service masato-hello mydb
$ cf restage masato-hello
```

### 確認

curlコマンドを使って、CloudFoundryにデプロイしたアプリに接続します。残念ながら、組み込みのH2の接続情報が表示されてしまいます。

``` bash
$ curl masato-hello.mybluemix.net
<!DOCTYPE html>

<html>
  <head>
    <title>Cloud Services</title>
  </head>
  <body>
    <h2>Cloud Services</h2>
    <table class="table table-striped">
      <thead>
        <tr>
          <th>Service Connector Type</th>
          <th>Connection address</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td><strong>org.apache.tomcat.jdbc.pool.DataSource</strong></td>
          <td>&lt;bad url&gt; h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE</td>
        </tr>
      </tbody>
    </table>
  </body>
</html>
```

### Pivotal Web Servicesと比較

Pivotal Web Servicesへpushしたときのログでは以下でした。

``` bash
-----> Java Buildpack Version: v2.5 | https://github.com/cloudfoundry/java-buildpack.git#840500e
-----> Downloading Open Jdk JRE 1.8.0_25 from https://download.run.pivotal.io/openjdk/lucid/x86_64/openjdk-1.8.0_25.tar.gz (1.3s)
       Expanding Open Jdk JRE to .java-buildpack/open_jdk_jre (1.1s)
-----> Downloading Spring Auto Reconfiguration 1.5.0_RELEASE from https://download.run.pivotal.io/auto-reconfiguration/auto-reconfiguration-1.5.0_RELEASE.jar (0.1s)
```


Pivotal Web Servicesは以下の組み合わせになります。

* Java Buildpack Version: v2.5
* Open Jdk JRE 1.8.0_25
* Spring Auto Reconfiguration 1.5.0_RELEASE

一方、Bluemixの場合は以下のような組み合わせになります。同じJARを使っても、Spring Cloudを使ってauto-reconfigurationをしてくれないようです。

* Liberty Buildpack Version: v1.9
* IBM 1.7.1 JRE
* Auto Reconfiguration 1.5.0_RELEASE


### まとめ

Liberty BuildpackでもSpring Auto Reconfigurationが入るように調べていこうを思います。

* Java8でなく、IBM 1.7.1 JREを使っている
* Spring Auto Reconfigurationがdropletに入らないので、デフォルトでSpring Cloudが使えない
* 自動的に@Profile("cloud")を判断して有効にしてくれない

