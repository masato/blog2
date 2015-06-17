title: "Packerを使いWindows上でOVAを作成する - Part2: VMware Workstation ビルド実行"
date: 2014-06-08 10:52:43
tags:
 - PackerOVA
 - Packer
 - Go
 - Windows
 - OVA
 - VMwareWorkstation
description: Part1でWindows上に構築したPackerを使ってOVAを作成します。Box-Cutterや時雨堂のPacker Templatesを参考に、JSONでテンプレートを書きます。VMware固有の設定もあり、記述量は多くなるのでわかる範囲で書きます。今回はShell Provisionerを使いましたが、ProvisionerにはSaltやAnsibleも使えるので、Packerで本番用のOVAを作るまで、少しずつ勉強していこうと思います。VMware Workstationに同梱されているVMware PlayerにインポートしてOVAの起動確認まで今回は試そうと思います。
---
[PackerでOVA作成](http://masato.github.io/tags/PackerOVA/)シリーズです。
[Part1](/2014/06/08/packer-windows-vmware-iso-ova-buildenv/)でWindows上に構築したPackerを使ってOVAを作成します。
[Box-Cutter](https://github.com/box-cutter/ubuntu-vm)や[時雨堂](https://github.com/shiguredo/packer-templates)の[Packer Templates](http://www.packer.io/docs/templates/introduction.html)を参考に、JSONでテンプレートを書きます。

VMware固有の設定もあり、記述量は多くなるのでわかる範囲で書きます。
今回は`Shell Provisioner`を使いましたが、ProvisionerにはSaltやAnsibleも使えるので、Packerで本番用のOVAを作るまで、少しずつ勉強していこうと思います。

`VMware Workstation`に同梱されている`VMware Player`にインポートしてOVAの起動確認まで今回は試そうと思います。

<!-- more -->

### 設定ファイルの準備
Windows上でOVAを作成するプロジェクトを作ります。
      
``` bash
$ cd %HOMEPATH%\packer_apps\ubuntu1404
$ cd %HOMEPATH%\packer_apps>tree /F ubuntu1404
%HOMEPATH%\packer_apps\ubuntu1404
│  template.json
│
├─http
│      preseed.cfg
└─scripts
        base.sh
        cleanup.sh
```

`Packer Template`を記述します。Vagrantを使わない設定にしました。provisionersは最低限の処理です。
post-processorsでは、ovftoolを使ってVMXからOVAに変換します。

``` json %HOMEPATH%\packer_apps\ubuntu1404\template.json
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
        " netcfg/get_hostname=ubuntu  1404<wait>",
        " noapic<wait>",
        " preseed/url=http://&#123;&#123; .HTTPIP  &#125;&#125;:&#123;&#123; .HTTPPort  &#125;&#125;/preseed.cfg<wait>",
        " -- <wait>",
        "<enter><wait>"
      ],
      "boot_wait": "10s",
      "disk_size": 4096,
      "guest_os_type": "linux",
      "http_directory": "http",
      "iso_checksum": "01545fa976c8367b4f0d59169ac4866c",
      "iso_checksum_type": "md5",
      "iso_url": "http://releases.ubuntu.com/14.04/ubuntu-14.04-server-amd64.iso",
      "ssh_username": "packer",
      "ssh_password": "packer",
      "ssh_port": 22,
      "ssh_wait_timeout": "10000s",
      "shutdown_command": "echo 'packer'|sudo -S shutdown -P now",
      "vmx_data": {
        "memsize": "512",
        "numvcpus": "1",
        "cpuid.coresPerSocket": "1"
      }
    }
  ],
  "provisioners": [
    {
      "type": "shell",
      "execute_command": "echo 'packer' | &#123;&#123; .Vars &#125;&#125; sudo -E -S sh '&#123;&#123; .Path }}'",
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

[Preseed](https://wiki.debian.org/DebianInstaller/Preseed)を使ったサイレントインストールの設定ファイルを書きます。
``` bash %HOMEPATH%\packer_apps\ubuntu1404\http\preseed.cfg
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

[Shell Provisioner](http://www.packer.io/docs/provisioners/shell.html)で実行するシェルスクリプトを書きます。パスワードなしでsudoできるようにvisudoの簡単な記述です。

``` bash %HOMEPATH%\packer_apps\ubuntu1404\scripts\base.sh
sed -i -e '/Defaults\s\+env_reset/a Defaults\texempt_group=sudo' /etc/sudoers
sed -i -e 's/%sudo  ALL=(ALL:ALL) ALL/%sudo  ALL=NOPASSWD:ALL/g' /etc/sudoers
```

終了処理でもっといろいろ削除しないとOVAファイルが小さくならないのですが、とりあえずaptのcleanだけします。
``` bash %HOMEPATH%\packer_apps\ubuntu1404\scripts\cleanup.sh
apt-get -y autoremove
apt-get -y clean
```

### packer build

作成した`Packer Template`を使ってOVAをビルドします。だいたい20分くらいかかります。
初回実行時はISOをダウンロードしてくるので、実際にはもう少し時間がかかります。
`VMware Workstation`が自動的に起動して、インストール作業が進みます。

``` bash
$ cd %HOMEPATH%\packer_apps\ubuntu1404
$ packer validate template.json
vmware-iso output will be in this color.

==> vmware-iso: Downloading or copying ISO
    vmware-iso: Downloading or copying: http://releases.ubuntu.com/14.04/ubuntu-
14.04-server-amd64.iso
==> vmware-iso: Creating virtual machine disk
==> vmware-iso: Building and writing VMX file
==> vmware-iso: Starting HTTP server on port 8246
==> vmware-iso: Starting virtual machine...
==> vmware-iso: Waiting 10s for boot...
==> vmware-iso: Connecting to VM via VNC
==> vmware-iso: Typing the boot command over VNC...
==> vmware-iso: Waiting for SSH to become available...
==> vmware-iso: Connected to SSH!
==> vmware-iso: Provisioning with shell script: scripts/base.sh
    vmware-iso: [sudo] password for packer:
==> vmware-iso: Provisioning with shell script: scripts/cleanup.sh
    vmware-iso: Reading package lists... Done
    vmware-iso: Building dependency tree
    vmware-iso: Reading state information... Done
    vmware-iso: 0 upgraded, 0 newly installed, 0 to remove and 41 not upgraded.
==> vmware-iso: Gracefully halting virtual machine...
    vmware-iso: Waiting for VMware to clean up after itself...
==> vmware-iso: Deleting unnecessary VMware files...
    vmware-iso: Deleting: output-vmware-iso\vmware.log
==> vmware-iso: Cleaning VMX prior to finishing up...
    vmware-iso: Detaching ISO from CD-ROM device...
==> vmware-iso: Compacting the disk image
==> vmware-iso: Running post-processor: ovftool
    vmware-iso (ovftool): Executing ovftool with arguments: [--targetType=ova --
acceptAllEulas output-vmware-iso\packer-vmware-iso.vmx packer_vmware-iso_vmware.
ova]
    vmware-iso (ovftool): Opening VMX source: output-vmware-iso\packer-vmware-is
o.vmx
    vmware-iso (ovftool): Opening OVA target: packer_vmware-iso_vmware.ova
    vmware-iso (ovftool): Writing OVA package: packer_vmware-iso_vmware.ova
Transfer Completed
    vmware-iso (ovftool): Completed successfully
    vmware-iso (ovftool):
Build 'vmware-iso' finished.

==> Builds finished. The artifacts of successful builds are:
--> vmware-iso:
```

### ovftoolでOVAの確認

作成したOVAの詳細をovftoolを使って表示させます。
OVAを他の環境で動かす場合に問題になりそうな、PackerのVMwareドライバーが設定しているDisksとNICsを確認しておきます。

``` bash
$ ovftool packer_vmware-iso_vmware.ova
OVF version:   1.0
VirtualApp:    false
Name:          packer-vmware-iso

Download Size:  588.10 MB

Deployment Sizes:
  Flat disks:   40.00 GB
  Sparse disks: 1.60 GB

Networks:
  Name:        nat
  Description: The nat network

Virtual Machines:
  Name:               packer-vmware-iso
  Operating System:   *otherlinuxguest
  Virtual Hardware:
    Families:         vmx-09
    Number of CPUs:   1
    Cores per socket: 1
    Memory:           512.00 MB

    Disks:
      Index:          0
      Instance ID:    6
      Capacity:       40.00 GB
      Disk Types:     SCSI-lsilogic

    NICs:
      Adapter Type:   E1000
      Connection:     nat    
```
### VMware Playerで確認
`VMware Player`を起動して作成したOVAをインポートすると、デフォルトでは、`%HOMEPATH%\Documents\Virtual Machines`の下にフォルダが作成されVMXに展開されます。

コンソールから`username:packer`、`passwd:packer`でログインして仮想マシンが使える状態になったことを確認します。

### まとめ
`VMware Player`でOVAの動作確認までできました。
OVAファイルが他の環境でも動くことを確認したいので、VMwareを採用しているIDCFクラウドにプロビジョニングしてみます。

CIにDroneを使って、コミットされたらOVAをビルドしてテスト、successならイメージは定期的にクラウドストレージにputして、disposableなビルド環境を作りたいです。

今回はOVAでしたが、GCEの場合は10分単位の課金みたいなので、コミット後にイメージを毎回プロビジョニングしても、すぐ破棄すればそんなにコストがかからないのではと思っています。

GCEはインスタンスの起動が速いため、さくさくビルドとテストができそうな予感がして、次回試してみようと思います。



