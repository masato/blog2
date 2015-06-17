title: "Packerを使いWindows上でOVAを作成する - Part1: VMware Workstation ビルド環境"
date: 2014-06-08 05:32:40
tags:
 - Packer
 - PackerOVA
 - Go
 - Windows
 - OVA
 - VMwareWorkstation
description: この前OVAを公開してみんなに使ってもらう必要があったのですが、作成に時間がかかり手作業も多く面倒でした。OVAができたあとプロビジョニングしてみて、作業ミスをみつけると本当に心が折れます。Packerを使うとイメージ作成が自動化できて、GCEやDockerのイメージ作成も統一化したいので、これを機会に勉強してみます。さっそくPackerの使い方ググると、OSXのVagrantとVirtualBoxの使い方やPacker TemplatesはGitHubでもいくつかありました。ただ、今回はWindows上でVagrantを使わずにVMwareのOVAを作りたいので、なかなか情報がみつからず困ります。
---
[PackerでOVA作成](http://masato.github.io/tags/PackerOVA/)シリーズです。
この前OVAを公開してみんなに使ってもらう必要があったのですが、作成に時間がかかり手作業も多く面倒でした。
OVAができたあとプロビジョニングしてみて、作業ミスをみつけると本当に心が折れます。

[Packer](http://www.packer.io/)を使うとイメージ作成が自動化できるのでプロビジョニング前にテストができます。
GCEやDockerのイメージ作成も統一化したいので、これを機会に勉強してみます。

さっそくPackerの使い方ググると、OSXのVagrantとVirtualBoxの使い方や[Packer Templates](http://www.packer.io/docs/templates/introduction.html)はGitHubでもいくつかありました。
ただ、今回はWindows上でVagrantを使わずにVMwareのOVAを作りたいので、なかなか情報がみつからず困ります。

### TL;DR
WindowsのPackerでOVAを作る場合`VMWare Workstation`とovftoolが必要です。
また、ovftoolのpost-processorはGoをインストールしてexeをビルドする必要があります。

<!-- more -->

### VMware Workstationのインストール
[VMware Builder from ISO](http://www.packer.io/docs/builders/vmware-iso.html)を斜め読みしただけでWindowsでも`VMware Player`が使えると思ってしまい`packer build`しました。
すると`VMware application not found`とエラーになってしまいます。[ソースを読む](https://github.com/mitchellh/packer/blob/master/builder/vmware/common/driver_player5.go)と、driverのパスがLinuxになっています。

おかしいと思い、[VMware Builder from ISO](http://www.packer.io/docs/builders/vmware-iso.html)をよく読むと、`VMware Player on Linux`と書いてありました。
Windowsでは有償の`VMware Workstation`の9か10が必要のようです。

Goのコードは読みやすくて助かるのですが、自分で書くとちょっと物足りなく、最近はDartの方がおもしいです。

[ここ](http://www.vmware.com/jp/products/workstation/)から`VMware Workstation`をダウンロードしてインストールします。
`packer build`の初回起動時にライセンス入力を求められますが、スキップしても30日間は評価版で使えます。

### Packerのインストール

[ダウンロード](http://www.packer.io/downloads.html)ページから[Windows amd64](https://dl.bintray.com/mitchellh/packer/0.6.0_windows_amd64.zip)をダウンロードします。2014-06-08のバージョンは、0.6.0です。

arm版もあります。Goで書いているだけあってクロスコンパイルを簡単にしているようです。
インストールはとても簡単で、zipを解凍してPATHを通すだけです。シングルバイナリなので各exeファイルは10MBと大きめになっています。

今回は、すでにPATHを通してある`C:\usr\local\bin`に解凍しました。`Cygwin64 Terminal`を起動してインストールします。

``` bash
$ wget https://dl.bintray.com/mitchellh/packer/0.6.0_windows_amd64.zip
$ unzip -j 0.6.0_windows_amd64.zip -d /cygdrive/c/usr/local/bin/
```

バージョンを確認します。
``` bash
$ packer version
Packer v0.6.0
```

### Go1.2のインストール
Goは最新の1.2をインストールします。go.cryptoが1.1の場合はエラーになります。
```
undefined: cipher.AEAD
undefined: cipher.NewGCM
```

[ダウンロード](https://code.google.com/p/go/wiki/Downloads)ページから、Windows amd64の[インストーラー](https://storage.googleapis.com/golang/go1.2.2.windows-amd64.msi
)をダウンロードして実行します。

Goのバージョンを確認します。
``` bash
$ go version
go version go1.2.2 windows/amd64
```

GOPATHは、`C:\gocode`にしました。
``` bash
$ go env GOPATH
C:\gocode
$ go env GOROOT
C:\Go
```

GOPATHとGOROOTのbinをPATHに通します。
``` bash
C:\Go\bin;C:\gocode\bin
```


### go install用にGitとMercurialのインストール
`go install`コマンドを使うために、[Git](http://msysgit.github.io/)と[Mercurial](http://mercurial.selenic.com/wiki/Download)のインストーラーをダウンロードして実行します。
* [Gitインストーラー](https://github.com/msysgit/msysgit/releases/download/Git-1.9.2-preview20140411/Git-1.9.2-preview20140411.exe)
* [Mercurialインストーラー](https://bitbucket.org/tortoisehg/files/downloads/mercurial-3.0.1-x64.msi)

バージョンを確認します。
``` bash
$ git version
git version 1.9.2.msysgit.0
$ hg version
Mercurial Distributed SCM (version 3.0.1)
```

### packer-post-processor-ovftoolのインストール
[packer-post-processor-ovftool](https://github.com/iancmcc/packer-post-processor-ovftool)を`go install`します。

``` bash
$ go get github.com/iancmcc/packer-post-processor-ovftool
$ go install github.com/iancmcc/packer-post-processor-ovftool
```

### ovftoolのインストール
[VMware OVF Tool 3.5.1 for Windows 64 bit](https://www.vmware.com/support/developer/ovf/)をインストールします。
ダウンロードは`My VMware`から行うため、VMwareにアカウント登録が必要です。

デフォルトの場合以下にインストールされるので、PATHに通します。
``` bash
C:\Program Files (x86)\VMware\VMware Workstation\OVFTool
```

### まとめ
とりあえず開発環境のセットアップが終わりました。普段はUbuntuやOSX上で開発しているので、Windowsは面倒に感じます。`VMware Player`も同じ名前でLinuxとWindowsで機能が違うのも嵌まりました。

次は`Packer template`を買いてOVAをビルドしてみます。


