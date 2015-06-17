title: "Micro Services with Docker or OSv - Part2: ClojureをOSv上で動かす"
date: 2014-10-10 20:32:42
tags:
 - MicroServices
 - MicroOS
 - OSv
 - CoreOS
 - SmartOS
 - Clojure
 - OSX
 - Puppies-vs-Cattle
 - IDCFクラウド
description: CoreOS,SmartOS,OSvといったMicro OSの台頭はとても重要だと思います。さっそくIDCFクラウドで試してみようとOVA for VMWare ESXiをダウンロードしてデプロイしてみましたが失敗してしまいました。ovaをuntatarしてみるとovfのVirtualSystemTypeがvmx-08でです。PackerのOVAで確認したように、ESXi4にデプロイする場合はvmx-07にする必要があります。他にも嵌まりそうなので、とりあえずローカルのOSXでOSvを動かしてみようと思います。
---

CoreOS,SmartOS,OSvといった`Micro OS`の台頭はとても重要だと思います。さっそくIDCFクラウドで試してみようと[OVA for VMWare ESXi](http://downloads.osv.io.s3.amazonaws.com/cloudius/osv/osv-v0.13.esx.ova)をダウンロードしてデプロイしてみましたが失敗してしまいました。ovaをuntatarしてみるとovfのVirtualSystemTypeがvmx-08です。[PackerのOVA](http://masato.github.io/2014/06/22/packer-windows-idcf-cloud/)で確認したように、ESXi4にデプロイする場合はvmx-07にする必要があります。他にも嵌まりそうなので、とりあえずローカルのOSXでOSvを動かしてみようと思います。


<!-- more -->

### Micro ServicesとMicro OSについて

とても興味深いポストをいくつか読みました。PistonのJosshuaさんのPuppies-vs-Cattleのアナロジーは有名で、disposableなインフラを構築する上で常に意識しておく必要のある考え方です。レガシーなアプリはpuppyですが、サーバーに名前をつけていけません。

* [Kiss Your Bash Prompt Goodbye](http://pistoncloud.com/2013/10/kiss-your-bash-prompt-goodbye/)
* [The missing piece: Operating Systems for web scale Cloud Apps](http://blog.hendrikvolkmer.de/2013/10/11/the-missing-piece-operating-systems-for-web-scale-cloud-apps/)
* [Do we need a cloud orchestrator?](https://www.linkedin.com/today/post/article/20140819064341-12018337-do-we-need-a-cloud-orchestrator)

### Homebrewとhomebrew-caskのインストール

[Run Locally](http://osv.io/run-locally/)を読みながらOSXでOSvを動かす環境を作っていきます。

古いHomebrewが入っていたので、削除して再インストールします。

``` bash
$ cd /usr/local/Library && git stash && git clean -d -f
$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
$ brew --version
0.9.5
```

`brew doctor`を実行するとWarningがたくさんでます。

``` bash
$ brew doctor
Warning: You have uncommitted modifications to Homebrew
If this a surprise to you, then you should stash these modifications.
Stashing returns Homebrew to a pristine state but can be undone
should you later need to do so for some reason.
    cd /usr/local/Library && git stash && git clean -d -f
Warning: Your Xcode (5.1.1) is outdated
Please update to Xcode 6.0.1.
Xcode can be updated from the App Store.
```

Xcodeのアップデートなど、Warningの修正をしていきます。

``` bash
$ brew prune
Pruned 0 dead formulae
Pruned 54 symbolic links and 14 directories from /usr/local
$ sudo xcodebuild -license
$ brew link git
Linking /usr/local/Cellar/git/2.1.2... 207 symlinks created
```

再度doctorを実行すると、Warningが消えました。

``` bash
$ brew doctor
Your system is ready to brew.
```

homebrew-caskをインストールします。

``` bash
$ brew install caskroom/cask/brew-cask
```

### VirtualBoxインストール

homebrew-caskを使ってVirtualBoxをインストールします。

``` bash
$ brew cask install virtualbox
$ VBoxManage --version
4.3.16r95972
```

QEMUはbrewでインストールします。

```
$ brew install qemu
$ qemu-system-x86_64 --version
QEMU emulator version 2.1.2, Copyright (c) 2003-2008 Fabrice Bellard
```

### Capstan

[Capstan](http://osv.io/capstan/)はDockerfileに似た、OSvイメージのビルドツールです。
capstanコマンドは`$HOME/bin`にインストールされるのでPATHを通します。

```
$ echo $PATH
/Users/mshimizu/google-cloud-sdk/bin:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin
```

`~/.bash_profile`にPATHの追記をします。

``` bash ~/.bash_profile
export PATH=$HOME/bin:$PATH
```

Capstanを流行のワンライナーでインストールします。

``` bash
$ source ~/.bash_profile
$ curl https://raw.githubusercontent.com/cloudius-systems/capstan/master/scripts/download | bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   563  100   563    0     0    848      0 --:--:-- --:--:-- --:--:--   847
Downloading Capstan binary: http://osv.capstan.s3.amazonaws.com/capstan/v0.1.2/darwin_amd64/capstan
######################################################################## 100.0%
$ which capstan
/Users/mshimizu/bin/capstan
```


### ClojureをOSvで動かす準備

[Running Clojure on OSv: Easier With a New Capstan Template](http://osv.io/blog/blog/2014/07/27/capstan-lein-template/)を参考にして、ClojureをOSvで動かしてみます。


Javaをインストールします。homebrew-caskでインストールするとOracle Java 8がインストールされます。

``` bash
$ brew cask search java
==> Exact match
java
==> Partial matches
eclipse-java		 javafx-scene-builder
$ java -version
java version "1.8.0_20"
Java(TM) SE Runtime Environment (build 1.8.0_20-b26)
Java HotSpot(TM) 64-Bit Server VM (build 25.20-b23, mixed mode)
```

brewでLeiningenをインストールします。

``` bash
$ brew install leiningen
$ lein --version
Leiningen 2.5.0 on Java 1.8.0_20 Java HotSpot(TM) 64-Bit Server VM
```

### Leiningenでアプリ作成

leinのcapstanテンプレートを使いサンプルアプリを作成します。

``` bash
$ cd
$ lein new capstan new-app
Retrieving capstan/lein-template/0.1.0/lein-template-0.1.0.pom from clojars
Retrieving capstan/lein-template/0.1.0/lein-template-0.1.0.jar from clojars
Generating a fresh Capstan project.
$ cd new-app
```

作成されたClojureアプリです。

``` clj ~/new~app/src/new_app/core.clj
(ns new-app.core
  (:gen-class))

(defn -main [& args]
  (println "Hello from new-app: clojure on OSv project"))
```

### Capstanでイメージのビルドと実行

`capstan run`コマンドを使うと、ClojureアプリのOSvイメージをビルドしたあと、VirtualBoxでインスタンスが実行されます。

``` bash
$ capstan run
Building new-app...
Downloading cloudius/osv-openjdk/index.yaml...
149 B / 149 B [==========================================================================] 100.00 %
Downloading cloudius/osv-openjdk/osv-openjdk.vbox.gz...
70.85 MB / 70.85 MB [====================================================================] 100.00 %
Created instance: new-app
OSv v0.13
eth0: 10.0.2.15
Hello from new-app: clojure on OSv project
```
 
### capstanコマンド例

イメージの一覧です。

``` bash
$ capstan images
Name                           Description                                        Version                   Created
cloudius/osv-openjdk           OpenJDK 7/OSv base image for developers            v0.13                     2014-10-02T15:24:37
new-app
```

インスタンスの一覧です。

``` bash
$ capstan instances
Name                                Platform   Status     Image
new-app                             vbox       Stopped
```