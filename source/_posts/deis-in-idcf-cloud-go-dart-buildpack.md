title: "Deis in IDCFクラウド - Part3: GoとDartのBuildpack"
date: 2014-08-16 00:50:06
tags:
 - Deis
 - CoreOS
 - Go
 - Dart
 - Buildpack
description: Using Buildpacksによると、Deisはデプロイするアプリの言語を検知して適切なBuildpackを適用してくれそうです。Java, Ruby, Python, Node.js, Scala, Go, DartのBuildpackがバンドルされているので、私の使い方のは十分です。
---

[Using Buildpacks](http://docs.deis.io/en/latest/developer/using-buildpacks/)によると、Deisはデプロイするアプリの言語を検知して適切なBuildpackを適用してくれそうです。
Java, Ruby, Python, Node.js, Scala, Go, DartのBuildpackがバンドルされているので、私の使い方のは十分です。

<!-- more -->


### Deisにログイン

最初にDeisにログインします。

```
$ deis login http://deis.210.140.16.229.xip.io --username=masato
password:
Logged in as masato
```

### Goアプリのデプロイ

[example-go](https://github.com/deis/example-go)のサンプルを利用してみます。
`git clone`します。

```
$ mkdir -p ~/deis_apps
$ cd !$
$ git clone https://github.com/deis/example-go.git
$ cd example-go
```

Deisのリポジトリを作成します。

``` deis create
list of known hosts.
Git remote deis added
```

`git push`すると、ちゃんと`Go app detected`してくれました。これは便利便利。

```
$ git push deis master
Warning: Permanently added '[deis.210.140.16.229.xip.io]:2222,[210.140.16.229]:2222' (ECDSA) to the list of known hosts.
Counting objects: 28, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (23/23), done.
Writing objects: 100% (28/28), 10.04 KiB | 0 bytes/s, done.
Total 28 (delta 7), reused 0 (delta 0)
-----> Go app detected
-----> Installing go1.2.2... done
-----> Running: godep go install -tags heroku ./...
-----> Discovering process types
       Procfile declares types -> web
-----> Compiled slug size is 1.7M
remote: -----> Building Docker image
remote: Sending build context to Docker daemon 1.796 MB
remote: Sending build context to Docker daemon
remote: Step 0 : FROM deis/slugrunner
remote:  ---> f607bc8783a5
remote: Step 1 : RUN mkdir -p /app
remote:  ---> Running in 1a56be7a172d
remote:  ---> 99dcadb46001
remote: Removing intermediate container 1a56be7a172d
remote: Step 2 : ADD slug.tgz /app
remote:  ---> 8955dea5200c
remote: Removing intermediate container b9c3e991bdc3
remote: Step 3 : ENTRYPOINT ["/runner/init"]
remote:  ---> Running in 0bd7c94686b3
remote:  ---> 6e401be0aeb2
remote: Removing intermediate container 0bd7c94686b3
remote: Successfully built 6e401be0aeb2
remote: -----> Pushing image to private registry
remote:
remote:        Launching... done, v2
remote:
remote: -----> timely-zoologer deployed to Deis
remote:        http://timely-zoologer.deis.210.140.16.229.xip.io
remote:
remote:        To learn more, use `deis help` or visit http://deis.io
remote:
To ssh://git@deis.210.140.16.229.xip.io:2222/timely-zoologer.git
 * [new branch]      master -> master
```

curlで確認します。おなじみの`Powered by Deis`です。

```
$ curl -s http://timely-zoologer.deis.210.140.16.229.xip.io
Powered by Deis
```

### Dartアプリのデプロイ

これは予想以上に楽しいです。気をよくしてDartアプリもデプロイします。

Dartのbuildpackは、[heroku-buildpack-dart](https://github.com/igrigorik/heroku-buildpack-dart)

[example-dart](https://github.com/deis/example-dart)のサンプルを利用してみます。
`git clone`します。

```
$ cd ~/deis_apps
$ git clone https://github.com/deis/example-dart
$ cd example-dart
```

Deisにリモートリポジトリを作成します。

```
$ deis create
list of known hosts.
Git remote deis added
```

Goと同じように`git push`したのですが、エラーになってしまいました。
`Dart app detected`してくれましたが、`DART_SDK_URL`の指定が必要のようです。

```
$ git push deis master
Counting objects: 508, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (445/445), done.
Writing objects: 100% (508/508), 9.31 MiB | 0 bytes/s, done.
Total 508 (delta 33), reused 508 (delta 33)
-----> Dart app detected
-----> ENV_DIR is
-----> Welcome, this machine is: Linux 32fca63289f8 3.15.8+ #2 SMP Thu Aug 7 19:50:25 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
       ERROR: you must specify DART_SDK_URL to a Dart SDK for Linux. See README for this buildpack.
remote: Slugbuilder returned error code
To ssh://git@deis.210.140.16.229.xip.io:2222/jiggly-quadrant.git
 * [new branch]      master -> master
```

`DART_SDK_URL`を指定すると設定が始まります。

```
$ deis config:set DART_SDK_URL=https://github.com/selkhateeb/heroku-vagrant-dart-build/releases/download/latest/dart-sdk.tar
Creating config... done, v4

=== jiggly-quadrant
DART_SDK_URL: https://github.com/selkhateeb/heroku-vagrant-dart-build/releases/download/latest/dart-sdk.tar
```

deisリポジトリをdestroyしてから再作成します。

```
$ deis destroy --app=jiggly-quadrant --confirm=jiggly-quadrant
Destroying jiggly-quadrant... done in 0s
Git remote deis removed
$ deis create
$ deis config:set DART_SDK_URL=https://github.com/selkhateeb/heroku-vagrant-dart-build/releases/download/latest/dart-sdk.tar
```

再度`git push`します。`Dart app detected`の後に先ほどしていたURLからSDKをダウンロードしてくれます。

```
$ git push deis master
Counting objects: 508, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (445/445), done.
Writing objects: 100% (508/508), 9.31 MiB | 0 bytes/s, done.
Total 508 (delta 33), reused 508 (delta 33)
-----> Dart app detected
-----> ENV_DIR is
-----> Welcome, this machine is: Linux e9781e1a46c6 3.15.8+ #2 SMP Thu Aug 7 19:50:25 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
-----> Installing Dart VM via URL https://github.com/selkhateeb/heroku-vagrant-dart-build/releases/download/latest/dart-sdk.tar
remote:   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
remote:                                  Dload  Upload   Total   Spent    Left  Speed
remote: 100   341  100   341    0     0    434      0 --:--:-- --:--:-- --:--:--   434
remote: 100 11.5M  100 11.5M    0     0  1491k      0  0:00:07  0:00:07 --:--:-- 2498k
-----> Copy Dart binaries to app root
-----> Dart cmd found at -rwxr-xr-x 1 root root 11468640 Aug 15 11:05 /app/dart-sdk/bin/dart
remote: Dart VM version: 1.3.0 (Tue Apr 15 03:03:20 2014) on "linux_x64"
-----> Dart reports version:
       *** Found pubspec.yaml in /tmp/build/.
       *** Running pub get
       Pub 1.3.0
       Resolving dependencies... (3.2s)
       Downloading http_server 0.9.3...
       Downloading browser 0.10.0+2...
       Downloading mime 0.9.0+3...
       Downloading path 1.3.0...
       Got dependencies!
       *** Running pub build
       Building with "pub build"
       Loading source assets... (0.4s)
       Building basic_http_server... (0.1s)
       [Info from Dart2JS]:
       Compiling basic_http_server|web/index.dart...
       [Info from Dart2JS]:
       Took 0:00:05.727261 to compile basic_http_server|web/index.dart.
       Built 5 files to "build".
       total
-----> Discovering process types
       Procfile declares types -> web
-----> Compiled slug size is 12M
remote: Step 0 : FROM deis/slugrunner
remote:  ---> f607bc8783a5
remote: Step 1 : RUN mkdir -p /app
remote:  ---> Running in 5e0be8d4b848
remote:  ---> cafc89cb74e2
remote: Removing intermediate container 5e0be8d4b848
remote: Step 2 : ADD slug.tgz /app
remote:  ---> 09bff1aad3aa
remote: Removing intermediate container 18670a84972e
remote: Step 3 : ENTRYPOINT ["/runner/init"]
remote:  ---> Running in c73010bda870
remote:  ---> 3f2a306f6b77
remote: Removing intermediate container c73010bda870
remote: Successfully built 3f2a306f6b77
remote: -----> Pushing image to private registry
remote:
remote:        Launching... done, v3
remote:
remote: -----> madras-keypunch deployed to Deis
remote:        http://madras-keypunch.deis.210.140.16.229.xip.io
remote:
remote:        To learn more, use `deis help` or visit http://deis.io
remote:
To ssh://git@deis.210.140.16.229.xip.io:2222/madras-keypunch.git
 * [new branch]      master -> master
```

ログに表示されたURLにcurlで確認しますが、何も表示されません。

```
$ curl -s http://madras-keypunch.deis.210.140.16.229.xip.io
```

### Dartアプリのでバッグ

fleetctlからSSH接続してデバッグします。

``` 
$ fleetctl ssh madras-keypunch_v3.web.1.service
```

`CONTAINER ID`を確認します。

```
$ docker ps
CONTAINER ID        IMAGE                                   COMMAND                CREATED             STATUS              PORTS                           NAMES
bd81c5db48a1        10.1.2.34:5000/madras-keypunch:latest   /runner/init start w   22 minutes ago      Up 22 minutes       0.0.0.0:49155->5000/tcp         madras-keypunch_v3.web.1
```

nsenterでコンテナにアタッチします。

```
$ nse bd81c5db48a1
groups: cannot find name for group ID 11
root@bd81c5db48a1:/#
```
Dartアプリは/appディレクトリにあり、サーバープログラムは、`/bin/basic_http_server.dart`になります。

``` dart /app/bin/basic_http_server.dart
void main() {
  // Assumes the server lives in bin/ and that `pub build` ran
  var pathToBuild = join(dirname(Platform.script.toFilePath()), '..', 'build');
...
```

ルートパスは`/app/build`ですが、直下にindex.htmlはなく、`/app/build/web`に静的コンテンツがあります。

```
$ ls -al /app/build/web/
total 116
drwxr-xr-x 1 root root   112 Aug 15 11:06 .
drwxr-xr-x 1 root root     6 Aug 15 12:57 ..
-rw-r--r-- 1 root root 38859 Aug 15 11:06 index.dart.js
-rw-r--r-- 1 root root 72453 Aug 15 11:06 index.dart.precompiled.js
-rw-r--r-- 1 root root   253 Aug 15 11:06 index.html
drwxr-xr-x 1 root root    14 Aug 15 11:06 packages
```

正しいURLは、/webまたは、/web/index.htmlでした。


```
$ curl -s http://madras-keypunch.deis.210.140.16.229.xip.io/web/index.html
<!DOCTYPE html>

<html>
  <head>
    <title>Hello from Dart</title>
  </head>

  <body>
    <p id="text"></p>

    <script type="application/dart" src="index.dart"></script>
    <script src="packages/browser/dart.js"></script>
  </body>
</html>
```

index.dartというファイルもないので、index.dartをコピーします。

```
$ cp /app/web/index.dart /app/build/web/
```

DartiumからURLを開きます。これでサーバーサイドとクライアントサイドのDartアプリの実行を確認できました。

{% img center /2014/08/16/deis-in-idcf-cloud-go-dart-buildpack/hello-dart.png %}

