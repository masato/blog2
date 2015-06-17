title: "Packerを使いWindows上でOVAを作成する - Part10: Nested VT-x VMwarePlayerでOVA作成"
date: 2014-06-19 23:56:05
tags:
 - PackerOVA
 - VMwarePlayer
 - Packer
 - Ubuntu
 - IDCFクラウド
 - NestedVirtualization
description: Part9で構築したNested VT-x VMware Playerを動かしているWindows Ubuntu 14.04 Desktop amd64にログインします。今回の目的は、LinuxのVMware Playerに、VIXとVDDKを別途ダウンロードすることにより、PackerでOVAを作成するときに必要な`VMware Workstation`の機能と同等にします。
---

[PackerでOVA作成](http://masato.github.io/tags/PackerOVA/)シリーズです。
[Part9](/2014/06/16/packer-windows-nested-vmware-palyer/)で構築した`Nested VT-x VMware Player`を動かしている`Windows VMware Player`上の`Ubuntu 14.04 Desktop amd64`にログインします。

今回の目的は、Linuxの`VMware Player`に、VIXとVDDKを別途ダウンロードすることにより、
PackerでOVAを作成するときに必要な`VMware Workstation`と同等のコマンドを使えるようにすることです。

<!-- more -->

### VMware Playerの初期設定

`VMware Player`を起動して、SoftwareupdateとVMwareToolsダウンロードしておきます。
初回起動時に、VMwarePlayerがダイアログを出すのでに二度目以降は表示しないチェックを入れます。

Player Preference
* Close Behavior -> Confirm before closing a virtual machineにアンチェック
* Software Updates -> Check for product updates on start upにアンチェック
* Software Updates -> Check for software components as neededにアンチェック


### VIX SDK for Linux 64-bit

[VIX SDK for Linux 64-bit](https://my.vmware.com/jp/web/vmware/free#desktop_end_user_computing/vmware_player/6_0|PLAYER-602|drivers_tools)をダウンロードします。

今回のバージョンは1.13.2です。
インストールするとvmrunコマンドが使えるようになります。

``` bash
$ sudo bash VMware-VIX-1.13.2-1744117.x86_64.bundle
$ which vmrun
/usr/bin/vmrun
```

### VDDK SDK for Linux

[VDDK SDK for Linux](https://my.vmware.com/jp/group/vmware/details?downloadGroup=VDDK550&productId=353)のダウンロードにはVMwareアカウントが必要なのでログインします。
バージョンは5.5.0です。

``` bash
$ cd ~/Downloads
$ tar zxvf VMware-vix-disklib-5.5.0-1284542.x86_64.tar.gz
$ cd vmware-vix-disklib-distrib/
```

Perlのインストーラーを実行します。

``` bash
$ sudo ./vmware-install.pl
```

### Packer環境

Goなのでさっと構築します。

``` bash
$ cd ~/Downloads
$ wget https://dl.bintray.com/mitchellh/packer/0.6.0_linux_amd64.zip
$ unzip 0.6.0_linux_amd64.zip -d ~/bin
$ source ~/.profile
```

バージョンは0.6.0です。
``` bash
$ packer version
Packer v0.6.0
```

### Goの開発環境
PackerのProcessorをビルドするため、Goの開発環境を構築します。最低限マルチプレクサとvimを使えるようにします。

``` bash
$ sudo sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list
$ sudo apt-get update
$ sudo apt-get install byobu
```

/usr/bin/vim.basicを選択します。
``` bash
$ sudo apt-get install vim
$ sudo update-alternatives --config editor
```

Goのビルドに必要な、いつものセットです。
``` bash
$ sudo apt-get install golang git mercurial
$ echo export GOPATH=$HOME/gocode >> ~/.profile
$ echo export PATH=$PATH:$GOPATH/bin >> ~/.profile
$ source ~/.profile
```

### packer-post-processor-ovftool のインストール
Packerでovftoolを使いOVAを作成するpost-processorの[packer-post-processor-ovftool](https://github.com/iancmcc/packer-post-processor-ovftool)を`go install`します。
``` bash
$ go get github.com/iancmcc/packer-post-processor-ovftool
$ go install github.com/iancmcc/packer-post-processor-ovftool
```

~/.packerconfigにインストール設定を記述します。
``` json ~/.packerconfig
{
  "post-processors": {
    "ovftool": "packer-post-processor-ovftool"
  }
}
```

### ESXi4用OVAの作成プロジェクト

プロジェクトを作成します。
``` bash
$ mkdir -p ~/packer_apps/ubuntu1404
$ cd !$
$ mkdir http scripts
```

template.jsonを作成します。今回はESXi4を使っているIDCFクラウド用のOVAを作成するため、
vmx_dataに`"virtualHW.version": "7"`を指定します。あとISOは日本のミラーからダウンロードするようにしました。

``` json ./template.json
{
  "builders": [
    {
      "type": "vmware-iso",
      "boot_command": [
        "<esc><wait>",
        "<esc><wait>",
        "<enter><wait>",
        "/install/vmlinuz<wait>",
        " auto<wait>",
        " console-setup/ask_detect=false<wait>",
        " console-setup/layoutcode=us<wait>",
        " console-setup/modelcode=pc105<wait>",
        " debconf/frontend=noninteractive<wait>",
        " debian-installer=en_US<wait>",
        " fb=false<wait>",
        " initrd=/install/initrd.gz<wait>",
        " kbd-chooser/method=us<wait>",
        " keyboard-configuration/layout=USA<wait>",
        " keyboard-configuration/variant=USA<wait>",
        " locale=en_US<wait>",
        " netcfg/get_hostname=ubuntu-1404<wait>",
        " netcfg/get_domain=vagrantup.com<wait>",
        " noapic<wait>",
        " preseed/url=http://&#123;&#123; .HTTPIP &#125;&#125;:&#123;&#123; .HTTPPort &#125;&#125;/preseed.cfg<wait>",
        " -- <wait>",
        "<enter><wait>"
      ],
      "boot_wait": "30s",
      "disk_size": 4096,
      "guest_os_type": "ubuntu-64",
      "http_directory": "http",
      "iso_checksum": "01545fa976c8367b4f0d59169ac4866c",
      "iso_checksum_type": "md5",
      "iso_url": "http://ftp.jaist.ac.jp/pub/Linux/ubuntu-releases/trusty/ubuntu-14.04-server-amd64.iso",
      "ssh_username": "packer",
      "ssh_password": "packer",
      "ssh_port": 22,
      "ssh_wait_timeout": "1000s",
      "shutdown_command": "echo 'packer'|sudo -S shutdown -P now",
      "vmx_data": {
        "virtualHW.version": "7",
        "memsize": "512",
        "numvcpus": "1",
        "cpuid.coresPerSocket": "1"
      }
    }
  ],
  "provisioners": [
    {
      "type": "shell",
      "execute_command": "echo 'packer' | &#123;&#123; .Vars &#125;&#125; sudo -E -S sh '&#123;&#123; .Path &#125;&#125;'",
      "override": {
        "vmware-iso": {
          "scripts": [
            "scripts/base.sh",
            "scripts/cleanup.sh"
          ]
        }
      }
    }
  ],
  "post-processors": [{
     "type": "ovftool",
     "only": ["vmware-iso"],
     "format": "ova"
  }]
}
```

preseed.cfgはユーザー情報だけ変更しました。

``` cfg ./http/preseed.cfg
choose-mirror-bin mirror/http/proxy string
d-i base-installer/kernel/override-image string linux-server
d-i clock-setup/utc boolean true
d-i clock-setup/utc-auto boolean true
d-i finish-install/reboot_in_progress note
d-i grub-installer/only_debian boolean true
d-i grub-installer/with_other_os boolean true
d-i partman-auto-lvm/guided_size string max
d-i partman-auto/choose_recipe select atomic
d-i partman-auto/method string lvm
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true
d-i partman-lvm/device_remove_lvm boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true
d-i partman/confirm_write_new_label boolean true

# Default user
d-i passwd/user-fullname string packer
d-i passwd/username string packer
d-i passwd/user-password password packer
d-i passwd/user-password-again password packer
d-i passwd/username string packer

# Minimum packages (see postinstall.sh)
d-i pkgsel/include string openssh-server
d-i pkgsel/install-language-support boolean false
d-i pkgsel/update-policy select none
d-i pkgsel/upgrade select none

d-i time/zone string UTC
d-i user-setup/allow-password-weak boolean true
d-i user-setup/encrypt-home boolean false
tasksel tasksel/first multiselect standard, ubuntu-server
```
scriptsは2つだけです。
base.shでは、aptを日本のURLへの変更と、ユーザーのsudo設定をします。
``` bash ./scripts/base.sh
sudo sed -i~ -e 's;http://us.archive.ubuntu.com/ubuntu;http://ftp.jaist.ac.jp/pub/Linux/ubuntu;' /etc/apt/sources.list
sed -i -e '/Defaults\s\+env_reset/a Defaults\texempt_group=sudo' /etc/sudoers
sed -i -e 's/%sudo  ALL=(ALL:ALL) ALL/%sudo  ALL=NOPASSWD:ALL/g' /etc/sudoers
```
cleanup.shではaptのクリーンだけしました。
``` bash ./scripts/cleanup.sh
apt-get -y autoremove
apt-get -y clean
```


### packer build

ようやく`packer build`できます。その前に一応validateで構文チェックします。
``` bash
$ packer validate template.json
Template validated successfully.
$ PACKER_LOG=1 packer build template.json
```

作成されたOVAを確認します。OVAファイルはtarで展開します。

``` bash
$ tar xvf packer_vmware-iso_vmware.ova
packer_vmware-iso_vmware.ovf
packer_vmware-iso_vmware.mf
packer_vmware-iso_vmware-disk1.vmdk
```

OVFのVirtualSystemTypeは指定したようにvmx-07になっています。
``` xml packer_vmware-iso_vmware.ovf
        <vssd:VirtualSystemType>vmx-07</vssd:VirtualSystemType>
```

### マニフェストを削除してOVAの再パッケージ

ただしESXi4の場合、OVAにマニフェストがあるとプロビジョニングに失敗するので、
マニフェストを削除してOVAを作り直します。
``` bash
$ mkdir -p ~/packer_apps/tmp
$ cd ~/packer_apps
$ mv packer_vmware-iso_vmware-disk1.vmdk tmp
$ mv packer_vmware-iso_vmware-disk1.ovf tmp
$ cd tmp
$ tar cvf ../packer_vmware-iso_vmware_07_nomf.ova packer_vmware*
```

### まとめ
もう何回目かわかりませんが、古いESXi4用のOVAをPackerで作成することができました。
次回はいよいよIDCFクラウドにプロビジョニングしても大丈夫そうです。
