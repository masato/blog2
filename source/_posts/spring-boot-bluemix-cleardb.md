title: "Spring BootをBluemixにデプロイする - Part2: Java Buildpackを使う"
date: 2014-12-21 00:56:15
tags:
 - Bluemix
 - CloudFoundry
 - SpringBoot
 - SpringCloud
 - ClearDB
description: BluemixにSpring BootをデプロイしLiberty BuildpackとSQLDBサービスを使ったテストはうまくいきませんでした。Spring Cloudが機能せず、DataSourceがinjectionされません。PWS(Pivotal Web Services)のデプロイで成功したときと同じ条件の、Oracle JDK 8、Java Buildpack、ClearDBサービスの組み合わせに変更して試してみます。サンプルアプリはこちらです。
---

BluemixにSpring Bootを[デプロイし](/2014/12/13/spring-boot-bluemix-db2/)、Liberty  BuildpackとSQLDBサービスを使ったテストはうまくいきませんでした。Spring Cloudが機能せず、DataSourceがinjectionされません。PWS(Pivotal Web Services)のデプロイで成功したときと同じ条件の、Oracle JDK 8、Java Buildpack、ClearDBサービスの組み合わせに変更して試してみます。サンプルアプリは[こちら](https://github.com/masato/spring-cloud-sample)です。

<!-- more -->

### ClearDBサービスを作成する

[PWS(Pivotal Web Services)で作成した](/2014/12/10/spring-boot-pivotal-web-services-cleardb-deploy/)ときと同様にCleaDBのサービスを作成します。

``` bash
$ cf create-service cleardb spark masato-db
````

### MySQLのJDBCドライバを追加する

Bluemixの場合は、build.gradleのdependenciesにMySQLのJDBCドライバを追加する必要がありました。build.gradleを編集してMySQLのJDBCドライバを追加します。Liberty Buildpackを使わないのでJDKは1.8のままにします。

``` groovy ~/spring-cloud-sample/build.gradle
sourceCompatibility = 1.8
targetCompatibility = 1.8

dependencies {
...
    compile("mysql:mysql-connector-java:5.1.34")
...
}
```

### Java Buildpackを指定する

BluemixのJavaの場合、デフォルトだとLiberty Buildpackになります。manifest.ymlを作成し、Java Buildpackを明示的に指定します。

``` yaml ~/spring-cloud-sample/manifest.yml
---
applications:
- name: masato-hello
  memory: 512M
  instances: 1
  path: build/libs/helloconfig-0.0.1-SNAPSHOT.jar
  buildpack: https://github.com/cloudfoundry/java-buildpack.git
```

アクティブなProfileにcloudを明示的に指定しなくても、Spring Auto Reconfigurationが自動的に判断してくれます。

``` yaml ~/spring-cloud-sample/manifest.yml
env:
  SPRING_PROFILES_DEFAULT: cloud
```

JARファイルビルドをし直し、manifest.ymlを使いpushします。今度はPWSと同じようにJava BuildpackとSpring Auto Reconfigurationがダウンロードしてdropletが作成されました。

``` bash
$ ./gradlew clean build
$ cf delete masato-hello
$ cf push
Using manifest file /home/docker/spring-cloud-sample/manifest.yml
...
-----> Downloaded app package (25M)
-----> Java Buildpack Version: de1c8c8 | https://github.com/cloudfoundry/java-buildpack.git#de1c8c8
-----> Downloading Open Jdk JRE 1.8.0_25 from https://download.run.pivotal.io/openjdk/lucid/x86_64/o
penjdk-1.8.0_25.tar.gz (found in cache)
-----> Downloading Spring Auto Reconfiguration 1.5.0_RELEASE from https://download.run.pivotal.io/au
to-reconfiguration/auto-reconfiguration-1.5.0_RELEASE.jar (found in cache)
-----> Uploading droplet (64M)
...
```

pushしたアプリとClearDBサービスをbindして、アプリを再起動します。

```
$ cf bind-service masato-hello masato-db
$ cf restage masato-hello
```

### DataSourceを確認する

curlでDataSourceの表示を確認します。ClearDBサービスと連携できました。

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
          <td>mysql://us-cdbr-iron-east-01.cleardb.net:3306/ad_b47d6b3cdd49cc5</td>
        </tr>
      </tbody>
    </table>
  </body>
</html>
```
