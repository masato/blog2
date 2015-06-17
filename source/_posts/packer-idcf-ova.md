title: "Packerを使いWindows上でOVAを作成する - Part3: IDCFクラウド"
date: 2014-06-09 22:05:49
tags:
 - Packer
 - Go
 - OVA
 - IDCFクラウド
 - gox
description: Part1,Part2を通してPackerで作成したOVAをテストするため、IDCFクラウドにプロビジョニングしてみました。ローカルのVirtualBoxとVMware Playerで動作しているので大丈夫だと思いましたが、結果的にPackerのGoの良いソースリードリーディングになりました。おかげでPackerやgoxが実現するGoのコンパイル環境のよい勉強になり、OVAが動いたことよりうれしかったりします。ESXi4.1用のOVAをovftoolで作る場合、OVFはのVirtualSystemTypはvmx-07にして、マニフェストファイルも削除する必要があります。
---

[Part1](/2014/06/08/packer-windows-vmware-iso-ova-buildenv/),[Part2](/2014/06/08/packer-windows-vmware-iso-ova-build/)を通してPackerで作成したOVAをテストするため、IDCFクラウドにプロビジョニングしてみました。

ローカルのVirtualBoxと`VMware Player`で動作しているので大丈夫だと思いましたが、
結果的にPackerのGoの良いソースリードリーディングになりました。

おかげで[Packer](http://www.packer.io/)や[gox](https://github.com/mitchellh/gox)が実現するGoのコンパイル環境のよい勉強になり、OVAが動いたことよりうれしかったりします。

### TL;DR
ESXi4.1用のOVAをovftoolで作る場合、OVFはのVirtualSystemTypは`vmx-07`にして、
マニフェストファイルも削除する必要があります。

<!-- more -->

Packerの開発をWindows上でするのはもうやめようと思いました。
Goなのでシングルバイナリが普通に動いて楽しいのですが、
`VMware Player`がLinuxと違ったり、バックスラッシュの問題でISOをコピーできなかったり、
嵌まりどころが満載です。

### バージョン情報
OVAを作成したPackerの環境は以下です。

* OS: Windows7 
* Go: go1.2.2 windows/amd64
* Packer: v0.6.9
* iancmcc/packer-post-processor-ovftool: 6b2f234ca4
* `VMware Workstation`: 10.0.2
* `VMware Player`: 6.0.2
* VirtualBox: 4.3.12

Packerの`vmware-iso'builder`はデフォルトでイメージを作っている環境のバージョンを使うので、
プロビジョニング先のバージョンがカレントでない場合、互換性を気にする必要があります。

### VirtualBox用OVFの修正

まだovfの仕様がよくわからないのですが、今回の環境だと`VMware Workstation`で作成したOVAは、
VirtualBoxは`vmw:osType`がPackerデフォルトのotherだと動ず、
"ubuntu64Guest"と明示する必要がありました。
``` xml
<OperatingSystemSection ovf:id="94" vmw:osType="ubuntu64Guest">
```

### IDCFクラウド用OVFの修正

[ここ](http://www.idcf.jp/blog/cloud/vmimport/)に従い、IDCFクラウドはESXi4.1を使っているので、vmxのバージョンは07にします。
``` xml
<vssd:VirtualSystemType>vmx-07</vssd:VirtualSystemType>
```

### まとめ
DMTFがCIMでXMLスキーマを策定しているので、バリデーションができないか何か探してみようとおもいます。

OVAはそれなりにサイズが大きくファイルコピーや圧縮に時間がかかるので、手戻りがあると悲しくなります。

ようやく自動でOVAが作成できるようになりました。
Ansibleやsalt-masterlessでprovisionしたり、もっと細かいOSの設定をしてみます。

serverspecでテストしてdroneでCIできるモダンな開発環境をつくるまでが、このシリーズの目標です。

最近は環境構築ばかりしていて、なかなか本業のプログラマの帽子がかぶれないので。
