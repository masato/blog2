title: "Spring BootをPivotal Web Servicesにデプロイする - Part2: ローカルのDocker開発環境で実行する"
date: 2014-12-09 22:10:15
tags:
 - DockerDevEnv
 - SpringBoot
 - SpringCloud
 - PivotalWebServices
 - PivotalCF
 - CloudFoundry
 - Gradle
description: Spring Cloudの使い方を調べていると、よい記事がみつかりました。Introducing Spring Cloud Deploying a Spring boot application to Cloud Foundry with Spring-Cloud hello-spring-cloud これを参考にして、リポジトリにspring-cloud-sampleのサンプルを作成していきます。Javaアプリを実行可能なJARファイルにできると、DockerコンテナにはJVMがあればデプロイ可能になります。
---

Spring Cloudの使い方を調べていると、よい記事がみつかりました。

* [Introducing Spring Cloud](http://spring.io/blog/2014/06/03/introducing-spring-cloud)
* [Deploying a Spring boot application to Cloud Foundry with Spring-Cloud](http://www.javacodegeeks.com/2014/08/deploying-a-spring-boot-application-to-cloud-foundry-with-spring-cloud.html)

これを参考にして、リポジトリに[spring-cloud-sample](https://github.com/masato/spring-cloud-sample)のサンプルを作成していきます。Javaアプリを実行可能なJARファイルにできると、DockerコンテナにはJVMがあればデプロイ可能になります。

<!-- more -->

### spring-cloud-sample

CloudFoundryの[hello-spring-cloud](https://github.com/cloudfoundry-samples/hello-spring-cloud)を参考にして、DataSourceのクラス名と接続URLを表示する簡単なプログラムを作成しました。

`@Profile("cloud")`アノテーションを指定しているので、CloudFoundryにデプロイした時にだけこのDataSourceのBeanは作成されます。

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

ローカルで実行するときはapplication.propertiesで指定したDataSourceが使用されます。

``` bash ~/spring-cloud-sample/src/main/resources/application.properties
spring.datasource.platform: h2
```

### bundle.gradle

dependenciesに、`org.springframework.cloud:spring-cloud-cloudfoundry-connector`を追加しました。[spring-cloud-cloudfoundry-connector](https://github.com/spring-cloud/spring-cloud-connectors/tree/master/spring-cloud-cloudfoundry-connector)の中で、`VCAP_SERVICES`環境変数のJSONをJacksonを使いアンマーシャルして、MySQLへの接続情報をセットしてくれます。

``` groovy ~/spring-cloud-sample/build.gradle
buildscript {
    ext {
        springBootVersion = '1.1.9.RELEASE'
        springloadedVersion = '1.2.1.RELEASE'
        springCloudVersion = '1.1.0.RELEASE'
    }
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
        classpath("org.springframework:springloaded:${springloadedVersion}")
    }
}

apply plugin: 'java'
apply plugin: 'spring-boot' 

jar {
    baseName = 'helloconfig'
    version = '0.0.1-SNAPSHOT'
}
sourceCompatibility = 1.8
targetCompatibility = 1.8

repositories {
    mavenCentral()
}


dependencies {
    compile("org.springframework.boot:spring-boot-starter-thymeleaf")
    compile("org.springframework.boot:spring-boot-starter-data-jpa")
    compile("org.springframework.cloud:spring-cloud-spring-service-connector:${springCloudVersion}")
    compile("org.springframework.cloud:spring-cloud-cloudfoundry-connector:${springCloudVersion}")

    compile("com.h2database:h2")
    testCompile("org.springframework.boot:spring-boot-starter-test")
}

task wrapper(type: Wrapper) {
    gradleVersion = '2.2.1'
}
```

### profileをcloudに指定


CloudFoundryにデプロイした時は、[Spring Auto-reconfiguration Framework](https://github.com/cloudfoundry/java-buildpack/blob/master/docs/framework-spring_auto_reconfiguration.md)、[java-buildpack-auto-reconfiguration](https://github.com/cloudfoundry/java-buildpack-auto-reconfiguration)の中で、`environment.addActiveProfile(CLOUD_PROFILE);`をしてくれるので動的にprofileが"cloud"になります。

``` java 
package org.cloudfoundry.reconfiguration.spring;
public final class CloudProfileApplicationContextInitializer implements
        ApplicationContextInitializer<ConfigurableApplicationContext>, Ordered {

    private static final String CLOUD_PROFILE = "cloud";
...
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        if (this.cloudUtils.isInCloud()) {
            addCloudProfile(applicationContext);
        } else {
            this.logger.warning("Not running in a cloud. Skipping 'cloud' profile activation.");
        }
    }

    private void addCloudProfile(ConfigurableApplicationContext applicationContext) {
        ConfigurableEnvironment environment = applicationContext.getEnvironment();

        if (hasCloudProfile(environment)) {
            this.logger.fine("'cloud' already in list of active profiles");
        } else {
            this.logger.info("Adding 'cloud' to list of active profiles");
            environment.addActiveProfile(CLOUD_PROFILE);
        }
    }
```

### gradlewを使って新しい環境でアプリを実行する

このリポジトリにはgradlewを作成して追加しています。[dockerfile/java](https://registry.hub.docker.com/u/dockerfile/java/)を使った新しいコンテナ上で、すぐにGradleの環境を構築してアプリを実行することができます。

``` bash
$ docker run -it --rm dockerfile/java:oracle-java8 bash
$ git clone https://github.com/masato/spring-cloud-sample
$ cd spring-cloud-sample
$ ./gradlew
...
Welcome to Gradle 2.2.1.

To run a build, run gradlew <task> ...

To see a list of available tasks, run gradlew tasks

To see a list of command-line options, run gradlew --help

BUILD SUCCESSFUL

Total time: 24.178 secs
```

### ローカルで実行する

bootRunを実行します。

``` bash
$ ./gradlew bootRun
```

Dockerホストから、コンテナのアプリへcurlを使い接続します。

``` bash
$ curl $(docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" $CONTAINER_ID):8080
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
