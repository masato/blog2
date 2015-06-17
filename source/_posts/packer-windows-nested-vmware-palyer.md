title: 'Packerを使いWindows上でOVAを作成する - Part9: VMwarePlayerでNested VT-x'
date: 2014-06-16 20:08:55
tags:
 - PackerOVA
 - VirtualBox
 - VMwarePlayer
 - Packer
 - Ubuntu
 - NestedVirtualization
description: Part8でVirtualBox 4.3.12に構築したNested VMware Playerには残念ながら64bitのOVAが作成できませんでた。'guest_os_type' 'ubuntu-64'と指定すると、VMware Playerが起動しませんでした。VBoxManageとLVMの復習にはなりましたが。'guest_os_type' 'linux'と指定すると、This kernel requires an x86-64 CPU ,but only detected an i686 CPU.Unable to boot - please use a kernel appropriate for your CPU.と表示され、VMware Playerは起動しますが、Preseedが実行されずインストールが中断してしまいます。'guest_os_type''linux'と指定して、32bitのubuntu-14.04-server-i386.isoを利用した場合に、ようやくPackerがOVAを作ってくれました。疑問になったのでNested Virtualizationを調べていくと、Mac で仮想マシンの入れ子 (Nested Virtualization) をするにテスト結果がありました。Intel-VT/EPTを有効にしても、VirtualBox4では64bitのゲストOSが作成できなかったようです。VMware Player 6については書いてなかったので実験してみます。
---
[PackerでOVA作成](http://masato.github.io/tags/PackerOVA/)シリーズです。
[Part8](/2014/06/15/packer-windows-vagrant-nested-vmware-player/)で`VirtualBox 4.3.12`に構築した`Nested VMware Player`には残念ながら64bitのOVAが作成できませんでした。

``` JSON
"guest_os_type": "ubuntu-64"
```
と指定すると、`VMware Player`が起動しませんでした。VBoxManageとLVMの復習にはなりましたが。

``` JSON
"guest_os_type": "linux"
```
と指定すると、

``` bash
"This kernel requires an x86-64 CPU ,but only detected an i686 CPU.Unable to boot - please use a kernel appropriate for your CPU."
```
と表示され、`VMware Player`は起動しますが、Preseedが実行されずインストールが中断してしまいます。

``` JSON
"guest_os_type": "linux"
```

と指定して、32bitのubuntu-14.04-server-i386.isoを利用した場合に、ようやくPackerがOVAを作ってくれました。

疑問になったので`Nested Virtualization`を調べていくと、
[Mac で仮想マシンの入れ子 (Nested Virtualization) をする](http://momijiame.tumblr.com/post/21783590370/mac-nested-virtualization)にテスト結果がありました。 
Intel-VT/EPTを有効にしても、VirtualBox4では64bitのゲストOSが作成できなかったようです。
`VMware Player 6`については書いてなかったので実験してみます。

<!-- more -->


### Windows7にVMware Playerをインストール

最初に作業マシンのWindows7に`VMware Player 6.0.2`を肯定的にインストールします。

### Ubuntu 14.04 Desktop amd64の仮想マシンの作成

Windows7の`VMware Player`を起動して、仮想マシンを作成します。
[Ubuntu 14.04 Desktop amd64](http://ftp.jaist.ac.jp/pub/Linux/ubuntu-releases/14.04/ubuntu-14.04-desktop-amd64.iso)のISOを指定すると、簡易インストールできるようです。

簡易インストールでは、ウィザードでLinuxのユーザー名やパスワードを入力すると、インストールプロセスを自動的に行ってくれます。
これは便利です。とりあえず、以下のように指定しました。

* フルネーム: packer
* ユーザー名: packer
* パスワード: packer
* 仮想マシン名: ubuntu-14.04-desktop-amd64
* ディスク最大サイズ: 30GB

「ハードウェアをカスタマイズ」ボタンを押して、VT-xを有効にします。
* メモリ: 2048GB
* プロセッサコアの数: 2
* 優先モード: Interl VT-x/EPT または AMD-V/RVI
* `Interl VT-x/EPT または AMD-V/RVIを仮想化`: チェック
* CPU パフォーマンス カウンタを仮想化: チェック

インストール終了後、`Nested VT-x`が有効か確認します。`vmx`が表示されるので成功しているようです。

``` bash
$ grep vmx /proc/cpuinfo
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts mmx fxsr sse sse2 ss syscall nx rdtscp lm constant_tsc arch_perfmon pebs bts nopl xtopology tsc_reliable nonstop_tsc aperfmperf eagerfpu pni pclmulqdq vmx ssse3 cx16 pcid sse4_1 sse4_2 x2apic popcnt aes xsave avx f16c rdrand hypervisor lahf_lm ida arat epb xsaveopt pln pts dtherm tpr_shadow vnmi ept vpid fsgsbase smep
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts mmx fxsr sse sse2 ss syscall nx rdtscp lm constant_tsc arch_perfmon pebs bts nopl xtopology tsc_reliable nonstop_tsc aperfmperf eagerfpu pni pclmulqdq vmx ssse3 cx16 pcid sse4_1 sse4_2 x2apic popcnt aes xsave avx f16c rdrand hypervisor lahf_lm ida arat epb xsaveopt pln pts dtherm tpr_shadow vnmi ept vpid fsgsbase smep
```

### Ubuntu 14.04 Desktop amd64にVMware Playerをインストール

`Ubuntu 14.04 Desktop amd64`にログインします。
Firefoxを起動して`VMware Player for Linux 64-bit`をダウンロード && インストールします。

``` bash
$ cd ~/Downloads
$ subo bash VMware-Player-6.0.2-1744117.x86_64.bundle
```

ゲストOSへ`VMware tools`も一緒にインストールします。

だんだんわかりにくくなってきましたが、以下のような階層を作ろうとしています。
```
3. Ubuntu 14.04 Server amd64
  VMware Player 6.0.2 for Linux 64-bit
2. Ubuntu 14.04 Desktop amd64 (Nested VT-x/EPT)
  VMware Player 6.0.2 for Windows 64-bit 
1. Windows 7 64-bit
```

### Nested VT-xのテスト

`Ubuntu 14.04 Desktop amd64`にインストールした`VMware Player 6.0.2 for Linux 64-bit`に、
今度は[Ubuntu 14.04 Server amd64](http://ftp.jaist.ac.jp/pub/Linux/ubuntu-releases/14.04/ubuntu-14.04-server-amd64.iso)をインストールします。名前以外はデフォルトです。

* フルネーム: packer
* ユーザー名: packer
* パスワード: packer
* 仮想マシン名: ubuntu-14.04-server-amd64
* 「ハードウェアをカスタマイズ」は今回は行いません。

ゲストOSの`VMware tools`は同様にインストールします。
`VirtualBox 4.3.12`の時と違いインストーラーが途中で止まりません。インストール終了後に64bitOSを確認します。
``` bash
$ uname -a
Linux ubuntu 3.13.0-24-generic #46-Ubuntu SMP Thu Apr 10 19:11:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
```

### まとめ

`Nested VT-x`ができたような気がします。次回は、`Ubuntu 14.04 Desktop amd64`にPacker開発環境をインストールして、ローカルの`VMware Player 6.0.2`を使い、OVAを作成してみます。

これが成功したらOSX用に`VMware Fusion 6.0`を購入しようかと思います。
`VMware Fusion 1.0`を購入したのは、もう7年前ですか。。

