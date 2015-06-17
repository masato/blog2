title: 'Packerを使いWindows上でOVAを作成する - Part4: Nested ESXi5 on VirtualBox ESXi5 Vagrant'
date: 2014-06-11 22:25:06
tags:
 - Vagrant
 - VirtualBox
 - Windows
 - MinGW
 - Mintty
 - Packer
 - PackerOVA
 - NestedVirtualization
 - Git
description: Part3でESXi4用のOVAがきれいに作成できなかったので、VirualBox 4.3.12にESXi5のNested Virtualizationを構築して、PackerのESXi5ドライバーを使ってみます。Vagrant and Chef on Windowsを参考にWindows上でVagrantを動かすところまで構築します。また、Minttyを使いやすくして、Windows7からLinuxを操作を便利にします。
---
[PackerでOVA作成](http://masato.github.io/tags/PackerOVA/)シリーズです。
[Part3](/2014/06/09/packer-idcf-ova/)でESXi4用のOVAがきれいに作成できなかったので、`VirualBox 4.3.12`にESXi5の
[Nested Virtualization](http://en.wikipedia.org/wiki/Nested_virtualization)環境を構築して、Packerの[ESXi5ドライバー](https://github.com/mitchellh/packer/blob/master/builder/vmware/iso/driver_esx5.go)を使ってみます。

[Vagrant and Chef on Windows](http://qiita.com/ogomr/items/98a33f47f6ba050adac4)を参考にWindows上でVagrantを動かすところまで構築します。また、Minttyを使いやすくしてWindows7からLinuxを操作を便利にします。

<!-- more -->

### VirtualBoxのインストール

VirtualBoxは[ダウンロード](http://www.oracle.com/technetwork/server-storage/virtualbox/downloads/index.html#vbox)ページから、最新バージョンの[4.3.12](http://download.virtualbox.org/virtualbox/4.3.12/VirtualBox-4.3.12-93733-Win.exe)をダウンロードしてインストーラーを実行します。

以下の場所にインストールされます。
``` bash
C:\Program Files\Oracle\VirtualBox
```

### Vagrantのインストール

Vagrantは[ダウンロード](http://www.vagrantup.com/downloads.html)ページから、最新バージョンの[1.6.3](https://dl.bintray.com/mitchellh/vagrant/vagrant_1.6.3.msi)をダウンロードしてインストーラーを実行します。

インストールディレクトリは、以下の場所に指定します。

```
C:\opt\HashiCorp\Vagrant
```

Windows版のVagrantには、[MinGW](ttp://www.mingw.org/),[Mintty](https://code.google.com/p/mintty/),[RubyInstaller](http://rubyinstaller.org/)が同梱されています。

``` bash
C:\opt\HashiCorp\Vagrant\embedded
```

Windowsなので仕方なく再起動すると、PATHが通って使えるようになります。

``` bash
$ vagrant -v
Vagrant 1.6.3
$ ruby -v
ruby 2.0.0p353 (2013-11-22) [i386-mingw32]
```

### Mintty

Vagrantに同梱されているMinttyを起動して`vagrant up`したあと、SSHコマンドを使うので使いやすい設定をします。

Minttyは以下の場所にインストールされます。

``` bash
C:\opt\HashiCorp\Vagrant\embedded\bin\mintty.exe
```

起動後ホームディレクトリに移動できるように、ショートカット作成を作成して、プロパティのリンク先にloginを追加します。
``` bash
C:\opt\HashiCorp\Vagrant\embedded\bin\mintty.exe /bin/bash --login
```

[Windows8 Proにvargrantを入れてみる(その１)](http://hayachi617.blogspot.jp/2013/09/windows8-provargrant.html)を参考に、
作成したショートカットからMinttyを起動して、.minttyrcの設定をします。
``` bash
$ cat << __EOF__ > ~/.minttyrc
BoldAsFont=no
Locale=ja_JP
Charset=UTF-8
FontHeight=12
Columns=100
Rows=34
Transparency=medium
Term=xterm-256color
RightClickAction=paste
OpaqueWhenFocused=no
PgUpDnScroll=yes
__EOF__
```

### プロジェクトの作成

Vagrantfileを配置する作業ディレクトリを作成します。
``` bash
$ mkdir -p ~/vagrant_apps/ubuntu-14.04
$ cd !$
```

### boxのダウンロード

Vagrantのboxは、[Docker](/tags/Docker/)でお世話になっている、phusionの[open-vagrant-boxes](https://github.com/phusion/open-vagrant-boxes)を使用します。
``` bash
$ vagrant init phusion/ubuntu-14.04-amd64
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
```

リソースが少ないので、CPUコアを1に指定します。
``` ruby ~/vagrant_apps/ubuntu14.04/Vagrantfile
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--cpus", "1"]
  end
end
```

`vagrant up`しまず。
``` bash
$ vagran up
```

### MinGWのパッケージを追加

VagrantでインストールされるMinGWには、viやsshが含まれていないため、MinGWをインストールした後コピーして使います。

[MinGW](http://www.mingw.org/)から、mingw-get-setup.exeをダウンロードして実行します。以下の場所にインストールします。

``` dos
C://opt/MinGW
```

Minttyを起動してMinGWの中身を、`Vagrant/embedded`にコピーして不足しているパッケージを追加します。

``` bash
$ cp -r /c/opt/MinGW/. /c/opt/HashiCorp/Vagrant/embedded/
```
mingw-getは以下のディレクトリに配置されます。

``` bash
$ which mingw-get
C:\opt\HashiCorp\Vagrant\embedded\bin\mingw-get.EXE
```

mingw-getの設定ファイルを編集し、MSYSのPATHをVagrantにコピーした場所に指定します。

defaults.xml

``` xml C:\opt\HashiCorp\Vagrant\embedded\var\lib\mingw-get\data\defaults.xml
<!--
<sysroot subsystem="MSYS" path="%R/msys/1.0" />
-->
<sysroot subsystem="MSYS" path="/opt/HashiCorp/Vagrant/embedded" />
```

profile.xml

``` xml C:\opt\HashiCorp\Vagrant\embedded\var\lib\mingw-get\data\profile.xml
<!--
<sysroot subsystem="MSYS" path="%R/msys/1.0" />
-->
<sysroot subsystem="MSYS" path="/opt/HashiCorp/Vagrant/embedded" />
```

min-getの更新をします。

``` bash
$ cd /c/opt/HashiCorp/Vagrant/embedded/var/lib/mingw-get/data/
$ mingw-get update
$ mingw-get upgrade
```

sshやvimのパッケージを`mingw-get install`します。

``` bash
$ mingw-get install msys-openssl msys-openssh
$ mingw-get install msys-rsync msys-tar msys-wget
$ mingw-get install msys-vim
```

### .bash_profileと.bashrc

環境設定ファイルを作成します。
~/.bash_profileに、VirtualBoxのコマンドのパスを通します。

``` bash ~/.bash_profile
export PATH=/c/Program\ Files/Oracle/VirtualBox/:$PATH
```

vimのエイリアスを設定します。

``` bash ~/.bashrc
alias vi='vim'
```

環境設定ファイルを再読込します。
``` bash
$ source ~/.bash_profile
$ source ~/.bashrc
```


### SSHの設定

`vagrant ssh`でなく、MinGWのsshを使うため、`.ssh/config`に設定情報を追加します。

``` bash 
$  mkdir -p ~/.ssh
$ vagrant ssh-config --host trusty >> ~/.ssh/config
```

SSHでVagrantのVMにログインします。

``` bash
$ ssh trusty
Welcome to Ubuntu 14.04 LTS (GNU/Linux 3.13.0-24-generic x86_64)

 * Documentation:  https://help.ubuntu.com/
Last login: Tue Apr 22 19:47:09 2014 from 10.0.2.2
vagrant@ubuntu-14:~$
```

### Git for Windows

[Git for Windows](http://msysgit.github.io/)から最新バージョンの[1.9.4](https://github.com/msysgit/msysgit/releases/download/Git-1.9.4-preview20140611/Git-1.9.4-preview20140611.exe)をダウンロードしてインストールします。

``` bash
c:\opt\Git
```

MinGWから、WindowsにインストールしたGitを使えるようにするため、以下の設定をします。
* Adjusting your PATH environment
 * Use Git from Windows Command Prompt

* Choosing the SSH executable
 * Use OpenSSH

* Checieout as-is, commit Unix-style line endings

Minttyを起動して、gitの確認をします。

``` bash
$ which git
c:\opt\Git\cmd\git.EXE
```

### ディレクトリ構成

状況によって`VirtualBox VMs`の場所が、Windowsのホームディレクトリだったり、ルートにあったりしますが、
最終的に、ディレクトリ構成は以下のようにしました。

.vagrant.dは、Windowsのホームディレクトリにあります。

``` bash
C:\Users\masato\.vagrant.d
```
そのほかのVagrantやVirtualBoxの設定やboxは、MinGWのホームディレクトリにあります。

``` bash
C:\opt\HashiCorp\Vagrant\embedded\home\masato
$ ls -a
.  ..  .VirtualBox  .bash_history  .inputrc  .minttyrc  .ssh  VirtualBox VMs  vagrant_apps
```

### まとめ

Linuxを主に使っていると感じませんが、OSXやWindowsで作業しているとVagrantはとても便利です。

次回はVagrantのVirtualBoxへ、NestedのESXiをインストールした後、Packer作業用のUbuntuからOVAを作成してみます。




