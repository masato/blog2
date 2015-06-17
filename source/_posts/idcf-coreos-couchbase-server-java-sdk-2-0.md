title: "Couchbase on Docker - Part2: Java SDK 2.0 とGradleでサンプル作成"
date: 2014-12-23 00:05:04
tags:
tags:
 - CoreOS
 - Couchbase
 - CouchbaseServer
 - IDCFクラウド
 - Gradle
 - Java8
description: IDCFクラウドのCoreOSクラスタにCouchbase Server 3.0.1をインストールしました。Couchbase Serverのクラスタに接続するサンプルを書いてみます。ドキュメントサイトにあるJava SDK 2.0を使ってHello Couchbase Exampleを実行してみます。CoreOSの仮装マシンの1台にJava開発環境のコンテナを用意してGradleプロジェクトを作成します。
---

IDCFクラウドのCoreOSクラスタにCouchbase Server 3.0.1を[インストール](/2014/12/20/idcf-coreos-couchbase-install/)しました。Couchbase Serverのクラスタに接続するサンプルを書いてみます。ドキュメントサイトにある[Java SDK 2.0](http://docs.couchbase.com/developer/java-2.0/java-intro.html)を使って、[Hello Couchbase Example](http://docs.couchbase.com/developer/java-2.0/hello-couchbase.html)を実行してみます。CoreOSの仮装マシンの1台に、Docker Hub Registryから[masato/baseimage](https://registry.hub.docker.com/u/masato/baseimage/)をpullしてGradleでプロジェクトを作成します。

<!-- more -->

### Java開発環境

[dockerfile/java](https://registry.hub.docker.com/u/dockerfile/java/)をpullして使い捨てのコンテナを作ります。`$JAVA_HOME`を確認します。

``` bash
$ docker run -it --rm dockerfile/java:oracle-java8
$ echo $JAVA_HOME
/usr/lib/jvm/java-8-oracle
$ java -version
java version "1.8.0_25"
Java(TM) SE Runtime Environment (build 1.8.0_25-b17)
Java HotSpot(TM) 64-Bit Server VM (build 25.25-b02, mixed mode)
```

### サンプルプロジェクト

作成したサンプルプロジェクトは[simple-couchbase](https://github.com/masato/simple-couchbase)のリポジトリにあります。`git clone`したあとgradlewを実行すればGradleの実行環境ができあがります。

``` bash
$ git clone https://github.com/masato/simple-couchbase.git
$ cd simple-couchbase
$ ./gradlew
```

### CouchMain.java

Couchbase Serverに接続するアプリのMainクラスです。`CouchbaseCluster.create("10.3.0.193","10.3.0.7","10.3.0.210");`の様に、引数へCouchbase Serverクラスタの3台のIPアドレスを指定します。

``` java ~/simple-couchbase/src/main/java/example/CouchMain.java
package example;
import com.couchbase.client.java.Cluster;
import com.couchbase.client.java.CouchbaseCluster;
import com.couchbase.client.java.Bucket;
import com.couchbase.client.java.document.json.JsonObject;
import com.couchbase.client.java.document.Document;
import com.couchbase.client.java.document.JsonDocument;
public class CouchMain
{
    public static void main(String args[])
    {
        try {
            Cluster cluster = CouchbaseCluster.create("10.3.0.193","10.3.0.7","10.3.0.210");
            Bucket bucket = cluster.openBucket();
            JsonObject user = JsonObject.empty()
                .put("firstname", "Walter")
                .put("lastname", "White")
                .put("job", "chemistry teacher")
                .put("age", 50);
            JsonDocument doc = JsonDocument.create("walter", user);
            JsonDocument response = bucket.upsert(doc);
            JsonDocument walter = bucket.get("walter");
            System.out.println("Found: " + walter);
            cluster.disconnect();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### build.gradle

build.gradleには`apply plugin: 'application'`と`mainClassName = 'example.CouchMain'`を記述して、runタスクからMainクラスを実行できるようにします。dependenciesにはMavenからjava-client:2.0.2を取得して追加します。

``` groovy ~/simple-couchbase/build.gradle
apply plugin: 'java'
apply plugin: 'application'

mainClassName = 'example.CouchMain'

repositories {
    mavenCentral()
}

sourceCompatibility = 1.8
targetCompatibility = 1.8

dependencies {
    compile 'com.couchbase.client:java-client:2.0.2'
}
```

### ./gradlew run

Couchbase ServerにupsertしたJSONをgetして中身を確認することができました。

```
$ cd ~/simple-couchbase
$ ./gradlew clean run
...
Found: JsonDocument{id='walter', cas=255239600078706, expiry=0, content={"firstname":"Walter","job":"chemistry teacher","age":50,"lastname":"White"}}
```
