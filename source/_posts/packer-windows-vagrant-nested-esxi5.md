title: 'Packerを使いWindows上でOVAを作成する - Part5: Nested ESXi5 on VirtualBox インストール'
date: 2014-06-12 23:49:42
tags:
 - PackerOVA
 - Packer
 - VirtualBox
 - Windows
 - ESXi5
 - NestedVirtualization
description: Part4でWindows上にVirtualBox 4.3.12の環境ができたので、Installing ESXi in VirtualBoxを参考にしながら、NestedのESXi5をインストールします。VMware WorkStationは30日間の評価期間後は使えなくなりますが、VirtualBoxの場合は無償で使い続けることができます。今回はVirtualBox上にNested ESXi5の環境を構築します。

---
[PackerでOVA作成](http://masato.github.io/tags/PackerOVA/)シリーズです。
[Part4](/2014/06/11/packer-windows-vagrant-mingw-mintty/)でWindows上に`VirtualBox 4.3.12`の環境ができたので、
[Installing ESXi in VirtualBox](http://www.virtxpert.com/installing-esxi-in-virtualbox/)を参考にしながら、NestedのESXi5をインストールします。`VMware WorkStation`は30日間の評価期間後は使えなくなりますが、VirtualBoxの場合は無償で使い続けることができます。

今回はVirtualBox上に`Nested ESXi5`の環境を構築します。

<!-- more -->

### ESXi5のISOダウンロード
[VMware vSphere Hypervisor 5.5](https://my.vmware.com/jp/web/vmware/evalcenter)からログインして、
最新の`ESXi 5.5 Update 1 ISO image`をダウンロードします。
無償ライセンスキーを発行するとこの画面から確認できます。


### VirtualBoxにESXi5のNested VMを作成

VirtualBoxを起動して、ファイル -> 環境設定から、仮想マシンの作成フォルダーを確認します。
```
C:\opt\HashiCorp\Vagrant\embedded\home\masato\VirtualBox VMs
```

「新規」ボタンから以下の設定で仮想マシンを作成します。
* 名前: ESXi5
* タイプ: Linux
* バージョン: Linux 2.6/3.x (64-bit)
* メモリーサイズ: 4096MB RAM
* ハードドライブ: 仮想ハードドライブを作成する -> VDI -> 可変サイズ -> 8GB(デフォルト)

VMにメモリを4GB割り当てないと、ESXi5のインストールができません。
ハードドライブの設定が終わると、VirutalBoxのメイン画面に戻ります。ESXi5の名前のVMは電源オフ状態です。

### VMの設定変更

ESXi5のVMを選択して、「設定」ボタンから編集します。

* システム -> プロセッサー -> プロセッサー数2,  拡張機能: PAE/NXを有効化
* ストレージ -> 空 -> CD/DVDドライブ -> CDアイコン -> ESXi ISOをマウント
* ネットワーク -> アダプター1 -> ホストオンリーアダプター(VirtualBox Host-Only Ethrenet Adapter)

「起動」ボタンから仮想マシンを起動すると自動的にESXi5のインストールが開始します。
CPU数は2以上を指定しないと、ESXi5の起動ができません。

以下の点を注意しながらインストール作業を終了させます。

* rootのパスワード: password
* 右のCtrlキーでホストに戻る
* HARDWARE_VIRTUALIZATION WARNINGが出るけど気にしない
* F11で確認してインストール

最後に、`Installation Complet`が表示されたら、VirtualBoxマネージャーからのISOを取り出した後に、
仮想マシンを再起動します。

再起動後に仮想マシンのコンソールに、ホストのIPアドレスが表示されます。
今回は、以下のIPアドレスになります。
https://192.168.56.101/


### ESXi5のSSH接続の設定

ESXi5のコンソール画面からF2キーを押して、rootでログインして`Customize System`画面を表示します。
以下のように進んで、SSHを有効にします。

* F2でCustomize System/View Logs
* Troubleshooting More Option -> Enable SSH

Escを2回押してログアウトします。

### vSphrere Client

ESXi5を起動すると、Webサイトも起動しているので、必要な場合`vSphere Client`をダウンロードします。
今回はローカルのWindowsにインストールしません。

https://192.168.56.101/

### UbuntuのVMからSSH接続

[Part4](/2014/06/11/packer-windows-vagrant-mingw-mintty/)で作成したUbuntuのVMから、
ESXi5にSSH接続します。

Minttyから `vagrant up`をしたあと、SSHでUbuntuにログインします。
``` bash
$ vagrant up
$ ssh trusty
```

UbuntuからESXi5にログインします。
```
$ ssh root@192.168.56.101
$ uname -a
VMkernel localhost 5.5.0 #1 SMP Release build-1623387 Feb 21 2014 17:19:17 x86_64 GNU/Linux
```

### まとめ
VirtualBoxに、`Nested ESXi5`とPacker作業用のUbuntuが構築できました。

UbuntuにPackerをインストールして、ESXi5上でOVAを作成してみます。前回はWindowsのPackerから作業したので、
Cygwinやパスのバックスラッシュ問題があったようなので、Linuxからだとうまくできそうな気がします。
