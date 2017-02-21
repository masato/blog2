title: "Eclipse IoT の紹介 - Part2: macOS SierraにEclipse Neon.2をインストールする"
date: 2017-02-01 12:25:41
categories:
 - IoT
 - EclipseIoT
tags:
 - EclipseIoT
 - EclipseKura
 - Eclipse
description: Eclipse KuraにインストールするOSGiバンドル開発用にEclipse IDE for Java Developersをインストールします。
---


　[Eclipse Kura](http://www.eclipse.org/kura/)の[OSGi](https://www.osgi.org/)ベースのIoT Gatewayのためのフレームワークです。アプリはjarファイルにマニフェストやリソースファイルを追加してパッケージングしたOSGiバンドルをデプロイして使います。今回はEclipuse Kuraのアプリのプログラミング環境としてmacOS Sierraに[Eclipse IDE for Java Developers](https://www.eclipse.org/downloads/packages/eclipse-ide-java-developers/neon2)をインストールして簡単な初期設定を行います。

<!-- more -->

## Java 8

　macOSのパッケージ管理ツールの[Homebrew](http://brew.sh/)と[Homebrew Cask](https://caskroom.github.io/)からJava 8をインストールします。Oracleの[ダウンロードサイト]( http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)から直接パッケージをダウンロードすることもできます。


　まだHombrewを使ったことがない場合はインストールします。

```
$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

　インストール済みの場合は、古いHombrewとパッケージを更新して最新の状態にします。

```
$ brew update && brew cleanup
```

　[Homebrew Cask](https://caskroom.github.io/)をインストールします。

```
$ brew install caskroom/cask/brew-cask
```

　[versions](https://github.com/caskroom/homebrew-versions)を追加して`java`を検索します。

```
$ brew tap caskroom/versions
$ brew search java
```

　現在の`java`パッケージのバージョンは`1.8.0_112`です。

```
$ brew cask info java
java: 1.8.0_112-b16
https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
Not installed
```

　Java8をインストールします。

```
$ brew cask install java
```

　`1.8.0_112`のバージョンがインストールされました。

```
$ java -version
java version "1.8.0_112"
Java(TM) SE Runtime Environment (build 1.8.0_112-b16)
Java HotSpot(TM) 64-Bit Server VM (build 25.112-b16, mixed mode)
```

　以下のコマンドを実行すると`JAVA_HOME`の場所を確認できます。

```
$ /usr/libexec/java_home -v 1.8
/Library/Java/JavaVirtualMachines/jdk1.8.0_112.jdk/Contents/Home
```

　`~/.bashrc`に環境変数の`JAVA_HOME`を設定します。

```
$ echo 'export JAVA_HOME=`/usr/libexec/java_home -v 1.8`' >> ~/.bashrc
$ source ~/.bashrc
$ echo $JAVA_HOME
/Library/Java/JavaVirtualMachines/jdk1.8.0_112.jdk/Contents/Home
```

## Maven

　Javaアプリのビルドツールの[Maven](https://maven.apache.org/)をインストールします。MavenやGradle、stbなどVMの開発ツールをインストールする場合は[SDKMAN!](http://sdkman.io/)が便利です。


　SDKMAN!をインストールします。

```
$ curl -s get.sdkman.io | /bin/bash
```

　`sdk`コマンドからMavenをインストールします。

```
$ sdk install maven 
```

　`3.3.9`のバージョンがインストールされました。

```
$ mvn --version
Apache Maven 3.3.9 (bb52d8502b132ec0a5a3f4c09453c07478323dc5; 2015-11-11T01:41:47+09:00)
```

## Eclipse Neon.2 (4.6.2)

　macOS用のインストーラーを[ダウンロードサイト](http://www.eclipse.org/downloads/eclipse-packages/)から取得します。
　
![eclipse-downloads.png](/2017/02/01/eclipse-iot-eclipse-neon2-setup/eclipse-downloads.png)

　`eclipse-inst-mac64.tar.gz`ファイルから解凍した`Eclipse Installer`を実行します。以下のディレクトリの`Eclipse`を起動します。

```
~/eclipse/java-neon/Eclipse
```



## 初期設定

　複数人で開発する場合はEclipseの設定を統一しておくとバージョン管理でdiffが正確に取れます。以下の私の好みなのでチームの規約に合わせて設定を確認してください。

### ファイルのタブは空白にする

　Eclipseの環境設定メニューからJavaのFormatterに新しいプロファイルの作成します。適当に`Eclise [myproject]`としました。

```
Eclipse -> 環境設定 -> Java -> Code Style -> Formatter -> New -> Eclipse [myproject] 
```

![java-formatter.png](/2017/02/01/eclipse-iot-eclipse-neon2-setup/java-formatter.png)


　`Indentation`タブを以下の設定にします。

* Tab policy: Spaces only
* Indentation size: 4
* Tab size: 4


![java-formatter-indent.png](/2017/02/01/eclipse-iot-eclipse-neon2-setup/java-formatter-indent.png)


　Mavenで使うpom.xmlなどXMファイルも同様に設定します。

```
Eclipse -> 環境設定 -> XML -> XML Files -> Editor
```

* Indent using spaces: チェック
* Indentation size: 2 

![xml-formatter.png](/2017/02/01/eclipse-iot-eclipse-neon2-setup/xml-formatter.png)


### ファイルの末尾は空白除去する


 Java Code Styleに`Clean Up`に新しいプロファイルの作成します。

```
Eclipse -> 環境設定 -> Java -> Code Style -> Clean Up -> New -> Eclipse [myproject]
```

![java-cleanup.png](/2017/02/01/eclipse-iot-eclipse-neon2-setup/java-cleanup.png)


* Remove trailing whitespace: チェック
 
![java-cleanup-tailtrail.png](/2017/02/01/eclipse-iot-eclipse-neon2-setup/java-cleanup-tailtrail.png)

　既存のソースファイルから末尾の空白を除去する場合はこの`Clean UP`を実行します。

```
src -> 右クリック -> Source -> Clean UP
```

　ファイル保存時に末尾の空白を削除する場合は`Save Actions`を使います。

```
Eclipse -> 環境設定 ->　Java -> Editor -> Save Actions
```

![java-save-actions.png](/2017/02/01/eclipse-iot-eclipse-neon2-setup/java-save-actions.png)


* Perform the selected actions on save: チェック


　Additional actionsの設定を変更します。

```
Additional actions -> Configure
```

* Remove trailing whitespace：　チェック
* All lines: チェック


![java-save-actions-configure.png](/2017/02/01/eclipse-iot-eclipse-neon2-setup/java-save-actions-configure.png)

### テーマ

　Eclipseの外観を`Dark`テーマを変えたい場合は以下の設定をします。

```
Eclipse -> 環境設定 -> General -> Apperance
```

* Theme: Dark

![appearance-theme.png](/2017/02/01/eclipse-iot-eclipse-neon2-setup/appearance-theme.png)


## プラグイン

　EclipseのプラグインもOSGiバンドルとして配布されています。プラグインも好みですがとりあえず最初に必要なプラグインをいくつかインストールします。

### EGit プラグイン

　[EGit](http://www.eclipse.org/egit/)プラグインはEclipseからGitを操作することができます。EGitのダウンロードサイトを追加します。


```
Help -> Install New Software... -> Add
```

* Name: EGit
* Location: http://download.eclipse.org/egit/updates

　使いたいパッケージをチェックしてインストールします。

* Git integration for Eclipse
* Java implementation of Git


![egit.png](/2017/02/01/eclipse-iot-eclipse-neon2-setup/egit.png)


　EclipseからSSH経由でGitリポジトリを使う場合は秘密鍵の設定を行います。

```
Eclipse -> 環境設定 -> General -> Network Connections -> SSH2
```

* SSH2 home: ホームディレクトリの`~/.ssh`など
* Private keys: GitHubに登録してある公開鍵とペアの秘密鍵など


![general-ssh2.png](/2017/02/01/eclipse-iot-eclipse-neon2-setup/general-ssh2.png)

　Gitパースペクティブを開きます。

```
Window -> Persepective -> Open Persepective -> Other -> Git
```

### M2Eclipse (m2e) プラグイン

 [M2Eclipse](http://www.eclipse.org/m2e/)はEclipseからMavenコマンドを実行することができます。プラグインはインストール済みですが、ダウンロードサイトを追加しておきます。

```
Help -> Install New Software... -> Add
```

* Name: m2e
* Location: http://download.eclipse.org/technology/m2e/releases


### Maven SCM Handler for EGit

　[Maven SCM Handler for EGit](https://github.com/tesla/m2eclipse-egit)はm2eから直接`git clone`してのpom.xmlの依存関係を解決してくれます。

　パッケージエクスプローラーにプロジェクトをインポートするときのダイアログの中で、m2e Marketplaceからインストールします。

```
Package Explorer > Import > Maven > Check out Maven Projects from SCM > m2e Marketplace > m2e-egit
```

![m2e-marketplace.png](/2017/02/01/eclipse-iot-eclipse-neon2-setup/m2e-marketplace.png)


　Eclipsをリスタートしてインストール終了です。



### mToolkitプラグイン

　これから[Eclipse Kura Documentation](https://eclipse.github.io/kura/)を読みながらEclipseで簡単なOSGiのアプリをビルドしていこうと思います。準備としてEclipse Kura上で動作するOSGiコンテナにリモートから接続できる[mToolkit](http://dz.prosyst.com/pdoc/mBS_SDK_8.1/eclipse_plugins/mtoolkit/introduction.html)のプラグインをインストールします。[Bosch](http://www.bosch.com/en/com/home/index.php)グループの[ProSyst](http://www.prosyst.com/)が提供しているOSGi管理ツールの[mBS SDK](https://dz.prosyst.com/pdoc/mBS_SDK_8.1/getting_started/stepbystep.html)に含まれています。


```
Help -> Install New Software... -> Add
```

* Name: mtoolkit
* Location: http://mtoolkit-neon.s3-website-us-east-1.amazonaws.com


* Group items by category: チェックを外す
* mTooklit: チェック


### Plug-in Development Environment (PDE)　プラグイン

　[Plug-in Development Environment (PDE)](http://www.eclipse.org/pde/)はEclipseのプラグインを開発するためのツールです。OSGiプロジェクトの開発ではComponent Definition (component.xml)を作成するウィザードを利用します。

```
Help -> Install New Software...
```

 `type filter text`フィールドに`Eclipse plug-in`を入力して検索します。

* Work with: Neon - http://download.eclipse.org/releases/neon
* type filter text: Eclipse plug-in
* Eclipse Plug-in Development Environment: チェック


![plugin-install.png](/2017/02/01/eclipse-iot-eclipse-neon2-setup/plugin-install.png)


　これでEclipse KuraのOSGiバンドル開発のためのEclipse Neon.2のセットアップは終了です。次回はサンプルプロジェクトをビルドして動かして見ようと思います。
