title: 'Packerを使いWindows上でOVAを作成する - Part6: Nested ESXi5 on VirtualBox リモートビルド'
date: 2014-06-13 00:43:47
tags:
 - Packer
 - PackerOVA
 - Go
 - VirtualBox
 - Windows
 - ESXi5
 - NestedVirtualization
description: Part5で構築した環境を使って、PackerからリモートのESXi5上でOVAをビルドします。一度WindowsのPackerからリモートビルドして失敗したので、Ubuntuの仮想マシンを作業環境にして再チャレンジです。どこかでVT-xを有効にしてくれているVPSがあると便利なのですが。
---
[PackerでOVA作成](http://masato.github.io/tags/PackerOVA/)シリーズです。
[Part5](/2014/06/12/packer-windows-vagrant-nested-esxi5/)で構築した環境を使って、PackerからリモートのESXi5上でOVAをビルドします。
一度WindowsのPackerからリモートビルドして失敗したので、Ubuntuの仮想マシンを作業環境にして再チャレンジです。

どこかでVT-xを有効にしてくれているVPSがあると便利なのですが。

### TL;DR

Ubuntuからもリモートの`Nested ESXi5`にISOのアップロードに失敗してしまい、ビルドができません。なぜだろう。。

<!-- more -->

### Vagrant

Ubuntuの仮想マシンにログインします。Vagrantの[phusion/open-vagrant-boxes](https://github.com/phusion/open-vagrant-boxes)から、"phusion-open-ubuntu-14.04-amd64を使います。

Minttyから `vagrant up`をしたあと、SSHでUbuntuにログインします。

``` bash
$ vagrant up
$ ssh trusty
```

### ESXi5

UbuntuからESXi5にログインします。

```
$ ssh root@192.168.56.101
```

packer buildでメッセージが表示されるので、GuestIPHackをします。

```
GuestIPHack is required, enable by running this on the ESX machine:
```

``` bash
$ esxcli system settings advanced set -o /Net/GuestIPHack -i 1
```

vim-cmdを使い、ESXi5のデータストアを確認します。` name = "datastore1"`をtemplate.jsonで使います。

``` bash
$ vim-cmd hostsvc/datastore/listsummary
(vim.Datastore.Summary) [
   (vim.Datastore.Summary) {
      dynamicType = <unset>,
      datastore = 'vim.Datastore:539bdf84-f652f3f9-ee0a-080027796f03',
      name = "datastore1",
      url = "/vmfs/volumes/539bdf84-f652f3f9-ee0a-080027796f03",
      capacity = 536870912,
      freeSpace = 520093696,
      uncommitted = 0,
      accessible = true,
      multipleHostAccess = <unset>,
      type = "VMFS",
      maintenanceMode = <unset>,
   }
]
```

### UbuntuにPackerのインストール

/etc/apt/sources.listが、us.archiveになっているので、日本のミラーに変更します。

``` bash
$ sudo sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list
$ sudo apt-get update
```

必要なパッケージをインストールします。

```
$ sudo apt-get install unzip
```

[Packer](http://www.packer.io/downloads.html)から、[Linux amd64](https://dl.bintray.com/mitchellh/packer/0.6.0_linux_amd64.zip)のzipファイルをダウンロードします。

``` bash
$ wget https://dl.bintray.com/mitchellh/packer/0.6.0_linux_amd64.zip
```

zipを解凍して、適当な場所に配置してPATHを通します。すごく簡単

``` bash
$ mkdir ~/bin
$ unzip 0.6.0_linux_amd64.zip -d ~/bin
$ which packer
/home/vagrant/bin/packer
$ packer version
Packer v0.6.0
```

### プロジェクトの作成

``` bash
$ mkdir -p ~/packer_apps/ubuntu1404
$ cd !$
$ mkdir http scripts
```

### template.json

template.jsonはPackerが実行する設定ファイルです。

``` json ~/packer_apps/ubuntu1404/template.json
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
      "boot_wait": "10s",
      "disk_size": 4096,
      "vmdk_name": "disk",
      "disk_type_id": "thin",
      "tools_upload_flavor": "linux",
      "guest_os_type": "ubuntu-64",
      "http_directory": "http",
      "iso_checksum": "01545fa976c8367b4f0d59169ac4866c",
      "iso_checksum_type": "md5",
      "iso_url": "http://releases.ubuntu.com/14.04/ubuntu-14.04-server-amd64.iso",
      "remote_host": "192.168.56.101",
      "remote_datastore": "datastore1",
      "remote_username": "root",
      "remote_password": "password",
      "remote_type": "esx5",
      "ssh_username": "packer",
      "ssh_password": "packer",
      "ssh_port": 22,
      "ssh_wait_timeout": "1000s",
      "shutdown_command": "echo 'packer'|sudo -S shutdown -P now",
      "headless": "true",
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
  ]
}
```

今回はUbuntuのOVAを作成するため、Preseedの設定ファイルを作成します。
template.jsonで、`http_directory`に指定したディレクトリからHTTPで設定ファイルを読み込んでくれます。

``` cfg ~/packer_apps/ubuntu1404/http/preseed.cfg
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

provisionersはtemplate.jsonでshellを指定しているので、実行するシェルスクリプトを作成します。

``` bash ~/packer_apps/ubuntu1404/scripts/base.sh
sudo sed -i~ -e 's;http://archive.ubuntu.com/ubuntu;http://ftp.jaist.ac.jp/pub/Linux/ubuntu;' /etc/apt/sources.list

sed -i -e '/Defaults\s\+env_reset/a Defaults\texempt_group=sudo' /etc/sudoers
sed -i -e 's/%sudo  ALL=(ALL:ALL) ALL/%sudo  ALL=NOPASSWD:ALL/g' /etc/sudoers
```

簡単な終了処理を実装します。

``` bash ~/packer_apps/ubuntu1404/scripts/cleanup.sh
apt-get -y autoremove
apt-get -y clean
```

### packer buildに失敗

`packer validate`でtemplate.jsonの構文チェックをします。

``` bash
$ packer validate template.json
Template validated successfully.
```

`packer build`します。初回はUbuntuのインストールISOをダウンロードするため時間がかかります。
ダウンロードしたISOは`packer_cache`ディレクトリにキャッシュされます。

失敗してしまいました。。。

``` bash
$ packer build template.json
...
    vmware-iso: Download progress: 99%
    vmware-iso: Download progress: 100%
==> vmware-iso: Uploading ISO to remote machine...
==> vmware-iso: Error uploading file: Process exited with: 1. Reason was:  ()
==> vmware-iso: Deleting output directory...
Build 'vmware-iso' errored: Error uploading file: Process exited with: 1. Reason was:  ()

==> Some builds didn't complete successfully and had errors:
--> vmware-iso: Error uploading file: Process exited with: 1. Reason was:  ()

==> Builds finished but no artifacts were created.
```

### packer buildのデバッグ

`PACKER_LOG=1`をprefixしてビルドを実行します。
ログを見ると、ISOファイルのアップロードに失敗しているようです。

``` bash
$ PACKER_LOG=1 packer build template.json
...
2014/06/14 08:22:47 /home/vagrant/bin/packer-builder-vmware-iso: 2014/06/14 08:22:47 Beginning file upload...
2014/06/14 08:24:38 /home/vagrant/bin/packer-builder-vmware-iso: 2014/06/14 08:24:38 SCP session complete, closing stdin pipe.
2014/06/14 08:24:38 /home/vagrant/bin/packer-builder-vmware-iso: 2014/06/14 08:24:38 Waiting for SSH session to complete.
2014/06/14 08:24:38 /home/vagrant/bin/packer-builder-vmware-iso: 2014/06/14 08:24:38 non-zero exit status: 1
==> vmware-iso: Error uploading file: Process exited with: 1. Reason was:  ()
2014/06/14 08:24:38 ui error: ==> vmware-iso: Error uploading file: Process exited with: 1. Reason was:  ()
2014/06/14 08:24:38 ui: ==> vmware-iso: Deleting output directory...
==> vmware-iso: Deleting output directory...
2014/06/14 08:24:38 /home/vagrant/bin/packer-builder-vmware-iso: 2014/06/14 08:24:38 opening new ssh session
2014/06/14 08:24:38 /home/vagrant/bin/packer-builder-vmware-iso: 2014/06/14 08:24:38 starting remote command: rm -rf /vmfs/volumes/datastore1/output-vmware-iso
2014/06/14 08:24:38 /home/vagrant/bin/packer-builder-vmware-iso: 2014/06/14 08:24:38 remote command exited with '0': rm -rf /vmfs/volumes/datastore1/output-vmware-iso
2014/06/14 08:24:38 ui error: Build 'vmware-iso' errored: Error uploading file: Process exited with: 1. Reason was:  ()
Build 'vmware-iso' errored: Error uploading file: Process exited with: 1. Reason was:  ()
2014/06/14 08:24:38 /home/vagrant/bin/packer-command-build: 2014/06/14 08:24:38 Builds completed. Waiting on interrupt barrier...
2014/06/14 08:24:38 machine readable: error-count []string{"1"}

==> Some builds didn't complete successfully and had errors:
2014/06/14 08:24:38 ui error:
==> Some builds didn't complete successfully and had errors:
2014/06/14 08:24:38 machine readable: vmware-iso,error []string{"Error uploading file: Process exited with: 1. Reason was:  ()"}
2014/06/14 08:24:38 ui error: --> vmware-iso: Error uploading file: Process exited with: 1. Reason was:  ()
--> vmware-iso: Error uploading file: Process exited with: 1. Reason was:  ()
2014/06/14 08:24:38 ui:
==> Builds finished but no artifacts were created.

==> Builds finished but no artifacts were created.
2014/06/14 08:24:38 waiting for all plugin processes to complete...
2014/06/14 08:24:38 /home/vagrant/bin/packer-command-build: plugin process exited
2014/06/14 08:24:38 /home/vagrant/bin/packer-builder-vmware-iso: plugin process exited
2014/06/14 08:24:38 /home/vagrant/bin/packer-provisioner-shell: plugin process exited
```

ESXi5にSSHログインしてアップロードしたISOを確認します。

``` bash
$ ssh root@192.168.56.101
$ ls -al /vmfs/volumes/datastore1/packer_cache
total 508936
drwxr-xr-x    1 root     root           420 Jun 14 08:11 .
drwxr-xr-t    1 root     root          1400 Jun 14 08:24 ..
-rw-r--r--    1 root     root     522715136 Jun 14 08:24 616e30c4df43460f8b93c3b5a9efb08868eb9b4778c5f5014607a37095b91524.iso
```

Ubuntuの`packer_cache`にダウンロードしたISOと比べると、ファイルサイズが違うので、SCPのアップロードが終了していないようです。

``` bash
$ ls -al ~/packer_apps/ubuntu1404/packer_cache/
total 577548
drwxr-xr-x 2 vagrant vagrant      4096 Jun 14 07:59 .
drwxrwxr-x 5 vagrant vagrant      4096 Jun 14 08:13 ..
-rw-rw-r-- 1 vagrant vagrant 591396864 Jun 14 08:11 616e30c4df43460f8b93c3b5a9efb08868eb9b4778c5f5014607a37095b91524.iso
```

### まとめ

リモートのESXi5にISOのアップロードに失敗してしまい、ビルドができません。
気長にデバッグしていきます。。。



