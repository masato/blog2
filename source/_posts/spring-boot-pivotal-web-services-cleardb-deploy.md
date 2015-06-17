title: "Spring BootをPivotal Web Servicesにデプロイする - Part3: ClearDBに接続する"
date: 2014-12-10 22:18:00
tags:
 - SpringBoot
 - SpringCloud
 - ClearDB
 - PivotalWebServices
 - PivotalCF
 - CloudFoundry
 - Gradle
 - MicroServices
 - SpringCloudNetflix
description: ローカルのDockerコンテナのJVMにSpring Bootのアプリをデプロイしましたが、JVMとコンテナの相性の良さを感じます。Spring Boot と Spring Cloud Netflix と Cloud Foundryを使いMicro Servicesが実現できそうな気がしてきました。まずは簡単なところから、前回作成したspring-cloud-sampleを使い、AmazonRDS互換のClearDBと連携してみます。
---

[ローカルのDockerコンテナ](/2014/12/09/spring-boot-piroval-web-services-sample/)のJVMにSpring Bootのアプリをデプロイしましたが、JVMとコンテナの相性の良さを感じます。Spring Boot と Spring Cloud Netflix と Cloud Foundryを使いMicro Servicesが実現できそうな気がしてきました。まずは簡単なところから、[前回作成](/2014/12/09/spring-boot-piroval-web-services-sample/)した[spring-cloud-sample](https://github.com/masato/spring-cloud-sample)を使い、AmazonRDS互換のClearDBと連携してみます。

<!--  more -->

### cloudfoundry/cliのインストール

cliは、64bit版のdebパッケージをダウンロードします。

``` bash
$ curl -L -o cf-cli_amd64.deb  https://cli.run.pivotal.io/stable?release=debian64
```

dpkgコマンドでdebパッケージをインストールします。バージョンは6.7.0でした。

``` bash
$ sudo dpkg -i cf-cli_amd64.deb
(Reading database ... 46310 files and directories currently installed.)
Preparing to unpack cf-cli_amd64.deb ...
Unpacking cf-cli (6.7.0) over (6.7.0) ...
Setting up cf-cli (6.7.0) ..
```

[cloudfoundry/cli](https://github.com/cloudfoundry/cli)は、新しくGoで書かれているのでインストールはとても簡単です。

``` bash
$ which cf
/usr/bin/cf
```

### Pivotal Web Servicesにlogin

最初にcfコマンドから、Pivotal Web Servicesにloginします。

``` bash
$ cf login -a https://api.run.pivotal.io
```

### Pivotal Web Servicesにpush

プロジェクトにあるgradlewを使い、JARファイルをビルドします。

``` bash
$ git clone https://github.com/masato/spring-cloud-sample
$ cd spring-cloud-sample
$ ./gradlew clean build
$ ls build/libs/
helloconfig-0.0.1-SNAPSHOT.jar  build/libs/helloconfig-0.0.1-SNAPSHOT.jar.original
```

JARファイルをPivotal Web Servicesにpushしてデプロイします。この時点では、まだMySQLサービスをバインドしていないので起動に失敗します。{myapp}はアプリケーション名に置き換えてください。

``` bash
$ cf push {myapp} -p build/libs/helloconfig-0.0.1-SNAPSHOT.jar
...
-----> Java Buildpack Version: v2.5 | https://github.com/cloudfoundry/java-buildpack.git#840500e
-----> Downloading Open Jdk JRE 1.8.0_25 from https://download.run.pivotal.io/openjdk/lucid/x86_64/openjdk-1.8.0_25.tar.gz (1.3s)
       Expanding Open Jdk JRE to .java-buildpack/open_jdk_jre (1.1s)
-----> Downloading Spring Auto Reconfiguration 1.5.0_RELEASE from https://download.run.pivotal.io/auto-reconfiguration/auto-reconfiguration-1.5.0_RELEASE.jar (0.1s)
...
```


### CleaDBサービスインスタンスを作成

PivotalCFのドキュメントの[Managing Service Instances with the CLI](http://docs.run.pivotal.io/devguide/services/managing-services.html)を読みながら、サービスインスタンスの使い方を見ていきます。

 
MarketplaceからClearDBサービスを探します。

``` bash
$ cf marketplace | grep cleardb
```

CleaDBサービスのインスタンスを作成します。{mydb}はデータベース名に置き換えてください。

``` bash
$ cf create-service cleardb spark {mydb}
```

### CleaDBサービスのバインド

CleaDBサービスをアプリケーションをバインドします。

``` bash
$ cf bind-service {myapp} {mydb}
```
 
### 確認

curlコマンドを使って、CloudFoundryにデプロイしたアプリに接続します。ClearDBへの接続情報が表示されました。

``` bash
$ curl {myapp}.cfapps.io
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
          <td>mysql://xxx.cleardb.net:3306/xxx</td>
        </tr>
      </tbody>
    </table>
  </body>
</html>
```

CloudFoundryで使うDataSourceの指定は以下だけです。

``` java ~/spring-cloud-sample/src/main/java/helloconfig/CloudConfiguration.java
package helloconfig;

import javax.sql.DataSource;

import org.springframework.cloud.config.java.AbstractCloudConfig;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.context.annotation.Bean;

@Configuration
@Profile("cloud")
public class CloudConfiguration extends AbstractCloudConfig {
    @Bean
    public DataSource dataSource() {
        return connectionFactory().dataSource();
    }
}
```

[spring-cloud-cloudfoundry-connector](https://github.com/spring-cloud/spring-cloud-connectors/tree/master/spring-cloud-cloudfoundry-connector)の中で、`VCAP_SERVICES`環境変数のJSONをJacksonを使いアンマーシャルして、MySQLへの接続情報をセットしてくれます。


spring-cloud-cloudfoundry-connectorは、build.gradleのdependenciesに追加してあります。

``` groovy ~/spring-cloud-sample/build.gradle
...
dependencies {
    compile("org.springframework.boot:spring-boot-starter-thymeleaf")
    compile("org.springframework.boot:spring-boot-starter-data-jpa")
    compile("org.springframework.cloud:spring-cloud-spring-service-connector:${springCloudVersion}")
    compile("org.springframework.cloud:spring-cloud-cloudfoundry-connector:${springCloudVersion}")

    compile("com.h2database:h2")
    testCompile("org.springframework.boot:spring-boot-starter-test")
}
...
```