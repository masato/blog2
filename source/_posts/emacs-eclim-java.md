title: "EclimでJavaもEmacsからコーディングする"
date: 2017-08-03 07:32:05
 - Java
 - Emacs
 - Eclim
tags:
 - TreasureData
 - Kafka
description: Java開発ではローカルのWindowsやmacOSのIntelliJ IDEAやEclipseなどのIDEを利用しますが、Node.jsやPythonなどスクリプト言語の開発はVimやEmacsのエディタという方も多いと思います。Eclimを使うとJavaも同じようにエディタから開発をすることができます。クラウドの仮想マシンに開発環境を構築すればローカルの設定依存せずターミナルからSSH接続していつでも同じ開発ができます。
---

　Java開発ではローカルのWindowsやmacOSのIntelliJ IDEAやEclipseなどのIDEを利用しますが、Node.jsやPythonなどスクリプト言語の開発はVimやEmacsのエディタという方も多いと思います。Eclimを使うとJavaも同じようにエディタから開発をすることができます。クラウドの仮想マシンに開発環境を構築すればローカルの設定依存せずターミナルからSSH接続していつでも同じ開発ができます。

<!-- more -->


## 仮想マシン

　クラウドに仮想マシンを用意します。今回はUbuntu 16.04 LTS (Xenial Xerus)にJavaの開発環境を構築します。

```bash
$ cat /etc/os-release
NAME="Ubuntu"
VERSION="16.04.3 LTS (Xenial Xerus)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 16.04.3 LTS"
VERSION_ID="16.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
VERSION_CODENAME=xenial
UBUNTU_CODENAME=xenial
```

　パッケージを更新しておきます。

```bash
$ sudo apt-get update && sudo apt-get dist-upgrade -y
```

## Java

　Java 8のSDKをインストールします。OpenJDK以外でも構いません。

```bash
$ sudo apt-get install openjdk-8-jdk -y
$ java -version
openjdk version "1.8.0_131"
OpenJDK Runtime Environment (build 1.8.0_131-8u131-b11-2ubuntu1.16.04.3-b11)
OpenJDK 64-Bit Server VM (build 25.131-b11, mixed mode)
```

## Eclipse Oxygen

　EclimはEclipseに接続してエディタからEclipseの一部の機能を使えるようにします。2017年6月にリリースされた[Eclipse Oxygen (4.7)](https://www.eclipse.org/oxygen/)をダウンロードします。

```bash
$ wget http://ftp.yz.yamagata-u.ac.jp/pub/eclipse/technology/epp/downloads/release/oxygen/R/eclipse-java-oxygen-R-linux-gtk-x86_64.tar.gz
$ tar zxvf eclipse-java-oxygen-R-linux-gtk-x86_64.tar.gz
$ sudo mv eclipse /opt/
```


## Emacs

　Emacs24をインストールします。

```bash
$ sudo apt-get install emacs24-nox emacs24-el -y
$ emacs --version
GNU Emacs 24.5.1
```

### Cask

　Emacsのパッケージ管理のため[Cask](http://cask.github.io/)をインストールします。

```bash
$ curl -fsSL https://raw.githubusercontent.com/cask/cask/master/go | python
$ echo 'export PATH="$HOME/.cask/bin:$PATH"' >> ~/.bashrc
$ source ~/.bashrc
```

　`~/.emacs.d`ディレクトリに以下のような設定ファイルを用意します。

```bash
$ tree ~/.emacs.d
.emacs.d.bak/
├── Cask
├── init.el
└── inits
    ├── 00-keybindings.el
    ├── 01-menu.el
    ├── 02-files.el
    └── 08-eclim.el
```

　`init.el`は[init-loader](https://github.com/emacs-jp/init-loader)を使いファイルを分割します。`~/.emacs.d/inits/08-eclim.el`以外はお好みで利用してください。

* ~/.emacs.d/init.el

```~/.emacs.d/init.el
(require 'cask "~/.cask/cask.el")
(cask-initialize)

(require 'init-loader)
(setq init-loader-show-log-after-init nil)
(init-loader-load "~/.emacs.d/inits")
```

　Caskにインストールするパッケージを記述します。Java開発用に[eclim](https://github.com/emacs-eclim/emacs-eclim)と[company-emacs-eclim](https://github.com/senny/emacs-eclim/blob/master/company-emacs-eclim.el)をインストールします。

* ~/.emacs.d/Cask

```
(source gnu)
(source melpa)

(depends-on "cask")
(depends-on "init-loader")

;; java
(depends-on "eclim")
(depends-on "company-emacs-eclim")
```

　Eclimの設定は[README](https://github.com/emacs-eclim/emacs-eclim/blob/master/README.md)に従います。ポップアップダイアログとコード補完機能は[company-mode](https://github.com/company-mode/company-mode)を使います。

* ~/.emacs.d/inits/08-eclim-el

```~/.emacs.d/inits/08-eclim.el
(require 'eclim)

;; enable eclim-mode globally
(setq eclimd-autostart t)
(global-eclim-mode)

;; Eclipse installation
(custom-set-variables
  '(eclim-eclipse-dirs '("/opt/eclipse/eclipse"))
  '(eclim-executable "/opt/eclipse/eclim"))

;; Displaying compilation error messages in the echo area
(setq help-at-pt-display-when-idle t)
(setq help-at-pt-timer-delay 0.1)
(help-at-pt-set-timer)

;; Configuring company-mode
(require 'company)
(require 'company-emacs-eclim)
(company-emacs-eclim-setup)
(global-company-mode t)
```

　以下は必須ではありませんがEmacsでよく使う設定です。`C-h`でバックスペースのキーバインドを変更します。

* ~/.emacs.d/inits/00-keybindings.el

```~/.emacs.d/inits/00-keybindings.el
(define-key global-map "\C-h" 'delete-backward-char)
(define-key global-map "\M-?" 'help-for-help)
```

　Emacsのメニューを非表示にします。

* ~/.emacs.d/inits/01-menu.el

```~/.emacs.d/inits/01-menu.el
(menu-bar-mode 0)
```

　行末の空白削除、バックアップを作らない、タブ設定などです。

* ~/.emacs.d/inits/02-files.el

```~/.emacs.d/inits/02-files.el
(when (boundp 'show-trailing-whitespace)
      (setq-default show-trailing-whitespace t))

(add-hook 'before-save-hook 'delete-trailing-whitespace)

(setq backup-inhibited t)
(setq next-line-add-newlines nil)
(setq-default tab-width 4 indent-tabs-mode nil)

(setq default-major-mode 'text-mode)
```

　最後に`cask`コマンドを実行してパッケージをインストールします。

```bash
$ cd ~/.emacs.d
$ cask install
```

## Eclim

 [Installing on a headless server](http://eclim.org/install.html#install-headless)にあるようにXサーバーが必要なEclipseをヘッドレスで利用するため仮想フレームバッファの[Xvfb](https://www.x.org/releases/X11R7.7/doc/man/man1/Xvfb.1.xhtml)をインストールします。

```bash
$ sudo apt-get install xvfb build-essential -y
```

　Eclipseのディレクトリに移動してEclimをインストールします。

```bash
$ cd /opt/eclipse
$ wget https://github.com/ervandew/eclim/releases/download/2.7.0/eclim_2.7.0.jar
$ java -Dvim.files=$HOME/.vim -Declipse.home="/opt/eclipse" -jar eclim_2.7.0.jar install
```

　 Xvfbとeclimdを起動します。

```bash
$ Xvfb :1 -screen 0 1024x768x24 &
$ DISPLAY=:1 /opt/eclipse/eclimd -b
```


## サンプルプロジェクト

　[maven-archetype-quickstart](http://maven.apache.org/archetypes/maven-archetype-quickstart/)のアーキタイプからサンプルプロジェクトを作成しEclimの動作確認をします。最初に[SDKMAN!](http://sdkman.io/)から[Maven](https://maven.apache.org/)をインストールします。

```bash
$ curl -s get.sdkman.io | /bin/bash
$ source ~/.sdkman/bin/sdkman-init.sh
$ sdk install maven
...
Setting maven 3.5.0 as default.
```

　適当なディレクトリで`mvn archetype:generate`を実行します。Eclipseの設定ファイルも`mvn eclipse:eclipse`で作成します。

```bash
$ mkdir ~/java_apps && cd ~/java_apps
$ mvn archetype:generate \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false \
  -DgroupId=com.example \
  -DartifactId=spike
$ cd spike
$ mvn eclipse:eclipse
```

　Emacsを起動します。

```bash
$ emacs
```

　アーキタイプから生成したプロジェクトをEclipseにインポートします。

```
M-x eclim-project-import
```

　ミニバッファでProject Directoryを聞かれるのでプロジェクトのディレクトリを指定します。

```　
Project Directory: ~/java_apps/spike/
```

　`Hello world!`に日付を追加しました。

* ~/java_apps/spike/src/main/java/com/example/App.java

```java ~/java_apps/spike/src/main/java/com/example/App.java
package com.example;
import java.util.Date;

/**
 * Hello world!
 *
 */
public class App
{
    public static void main( String[] args )
    {
        Date date = new Date();
        System.out.println( "Hello World!: " + date.getMinutes());
    }
}
```

　`date.get`まで入力するとポッップアップとミニバッファに候補が表示されます。カーソルで`getMinutes`を選択してエンターを押します。

![eclim.png](/2017/08/03/emacs-eclim-java/eclim.png)

　`C-x C-s`でファイルを保存すると`Eclim reports 0 errors, 1 warnings.`と表示されます。`getMinutes`にアンダーラインが引かれそこにカーソルを移動するとミニバッファにエラーメッセージが出ます。Hello worldなのでここでは無視します。

![eclim2.png](/2017/08/03/emacs-eclim-java/eclim2.png)

　mainメソッドがあるJavaファイルをバッファに表示して`eclim-run-class`を実行するとコンパイルとmainメソッドが走ります。

```
M-x eclim-run-class
```

![eclim3.png](/2017/08/03/emacs-eclim-java/eclim3.png)