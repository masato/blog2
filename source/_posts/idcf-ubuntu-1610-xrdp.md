title: "IDCFクラウドのUbuntu 16.04を16.10にupgradeしてxrdpリモートデスクトップ環境を構築する"
date: 2017-02-04 15:14:04
tags:
 - Ubuntu
 - MATE
 - IDCFクラウド
description: IDCFクラウドにUbuntu 16.10 MATEのデスクトップ環境を構築します。
---


　クラウド上にLinuxのデスクトップ環境を構築します。プログラミング言語によってはSSH接続してEmacsやVimなどのエディタで十分な場合もあります。JavaやScalaのプログラミングになると[Eclim](http://eclim.org/index.html)や[ENSIME](http://ensime.org/)といったCUIも使えますが、EclipseなどのGUIが使えると便利な場合があります。開発用にIDCFクラウド上にリモートデスクトップ環境を用意しようと思います。


<!-- more -->


## Ubuntu 16.04にボリュームを追加する

　最近の開発ではDockerを使うケースがほとんどですし、Mavenの`~/.m2`ディレクトリなどすぐに1GBを超えてしまいます。IDCFクラウドの標準ではルートディスクが15GBしかディスクがないため、面倒ですがデータディスクを増設して`/home`や`/var`にマウントして使います。

　Ubuntu 16.04のテンプレートの仮想マシンに100GBのボリュームを追加して起動します。`/dev/sda1`のルートディスクはext4でフォーマットされていました。今回はこのディスクを`/home`ディレクトリにマウントします。

```
# df -T
Filesystem     Type     1K-blocks    Used Available Use% Mounted on
udev           devtmpfs   4065756       0   4065756   0% /dev
tmpfs          tmpfs       816668    8912    807756   2% /run
/dev/sda1      ext4      15348720 1721424  12824584  12% /
tmpfs          tmpfs      4083324       0   4083324   0% /dev/shm
tmpfs          tmpfs         5120       0      5120   0% /run/lock
tmpfs          tmpfs      4083324       0   4083324   0% /sys/fs/cgroup
```

　増設した100GBのデータディスクは`/dev/sdb`のデバイスファイルです。

```
# fdisk -l
...
Disk /dev/sdb: 100 GiB, 107374182400 bytes, 209715200 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

　fdiskコマンドでパーティションを作成してext4でフォーマットします。n -> pと入力して残りはデフォルトでエンターキーを押します。

```
# fdisk /dev/sdb
...
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-209715199, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-209715199, default 209715199):

Created a new partition 1 of type 'Linux' and of size 100 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

　作成したパーティションにext4のファイルシステムを作成します。

```
# mkfs -t ext4 /dev/sdb1
```

　`/home`ディレクトリに作業ユーザーを作成します。`pwgen`コマンドを使いパスワードを生成します。`-B`フラグを指定するとパスワードとして判別が紛らわしい文字を除外して生成してくれます。

```
# apt-get update && apt-get install -y pwgen
# pwgen -B
doquevi9 zoh4ieY9 oqu4jooN EeFei7aV sha7Hiet aiha7Le4 Weehoo4a eixua7Ua
...
```

 `cloud-user`名でユーザーを作成します。管理者ユーザーとして仮想マシン作成時にrootユーザーの`/root/authorized_keys`に設定された鍵はコピーして使い、`sudo`もパスワードなして利用できるようにします。この辺りはセキュリティポリシーに従って設定してください。

```
# useradd -m -d /home/cloud-user -s /bin/bash cloud-user \
 && echo "cloud-user:doquevi9" | chpasswd \
 && mkdir -p /home/cloud-user/.ssh \
 && chmod 700 /home/cloud-user/.ssh \
 && cp /root/.ssh/authorized_keys /home/cloud-user/.ssh \
 && chmod 600 /home/cloud-user/.ssh/authorized_keys \
 && chown -R cloud-user:cloud-user /home/cloud-user/.ssh \
 && echo "cloud-user ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
```

　`/home`を移動するためのマウントポイントを作成して追加ディスクをマウントします。

```
# mkdir /mnt/home
# mount /dev/sdb1 /mnt/home
```

　`/home`を全て`/mnt/home`にコピーします。

```
# cp -a /home/* /mnt/home
```

　既存の`/home`ディレクトリはリネームしてバックアップしておきます。

```
# mv /home /home.old
# mkdir /home      
```

　`blkid`コマンドで`/dev/sdb1`のUUIDを調べます。

```
# blkid /dev/sdb1
/dev/sdb1: UUID="7b323902-0182-426a-8d76-991901d69c02" TYPE="ext4" PARTUUID="3986d957-01"
```

　`/etc/fstab`はバックアップを取ります。

```
$ cp -ip /etc/fstab /etc/fstab_`date "+%Y%m%d"`
```

　`/dev/sdb1`を`/home`へマウントする設定を追加します。

```bash:/etc/fstab
UUID=7b323902-0182-426a-8d76-991901d69c02 /home ext4 defaults 0  1
```

　rebootすると自動的に追加したデータディスクが`/home`ディレクトリにマウントして使えるようになります。

```
# reboot
```


## 16.04 LTSから16.10へupgradeする

　IDCFクラウドで利用できる最新のテンプレートはUbuntu Server 16.04 LTSです。[xrdp](http://www.xrdp.org/)は0.9から日本語キーボードが標準で使えるようになります。0.9はUbuntu 16.10から標準で利用できるので16.04から16.10へupgradeします。

　Ubuntuのupgradeは`do-release-upgrade`コマンドを利用します。またファイルの編集用にデフォルトエディタとして`vim`もインストールしておきます。
```
$ sudo apt-get update && sudo apt-get install -y update-manager-core vim
```

　`/etc/update-manager/release-upgrades`ファイルを`normal`に編集してLTS版からアップグレードできるようにします。

```bash:/etc/update-manager/release-upgrades
[DEFAULT]
#Prompt=lts
Prompt=normal 
```

　Ubuntu 16.04 LTSから16.10にupgradeします。

```
$ sudo do-release-upgrade
```

　SSH接続でupgradeをしているため途中で失敗して接続が切れる場合を考慮して、新しく別のポートでSSH接続しておきます。

```
This session appears to be running under ssh. It is not recommended
to perform a upgrade over ssh currently because in case of failure it
is harder to recover.

If you continue, an additional ssh daemon will be started at port
'1022'.
Do you want to continue?

Continue [yN]
```

　別のターミナルで1022ポートで接続を確認してからupgrade作業を継続します。

```
To continue please press [ENTER]
```

　upgradeには時間がかかります。`y`を押して作業を開始します。

```
Fetching and installing the upgrade can take several hours. Once the
download has finished, the process cannot be canceled.

 Continue [yN]  Details [d]
```

　upgrade作業中にいくつかダイアログが表示されます。デフォルトで以下のように選択しました。

* Postfix Configuration: No configuration
* Configuring grub-pc: keep the local version currently installed
* /etc/update-manager/release-upgrades: N
* Remove obsolete packages?: y
* System upgrade is complete.: y


　rebootするとupgrade作業は終了です。SSH接続し直してバージョンを確認します。

```
$ lsb_release -rd
Description:    Ubuntu 16.10
Release:        16.10
```

## xrdp

　Ubuntu 16.10にupgradeするとxrdpは0.9が使えるようになります。

```
$ sudo apt-get update
$ apt-cache show xrdp | grep Version
Version: 0.9.0~20160601+git703fedd-3
```

　xrdpをインストールして起動設定をします。

```　
$ sudo apt-get install xrdp -y
$ sudo systemctl enable xrdp.service
$ sudo systemctl start xrdp.service
```


### MATE

　クラウドでの利用に適した軽量なデスクトップ環境もいろいろありますが、GNOME2から派生した[MATE](http://mate-desktop.org/)が好みなので良く使います。

```
$ apt-cache show mate-core | grep Version
Version: 1.16.0+1
```

　バージョンは1.16です。

```
$ sudo apt update && sudo apt install mate-core mate-desktop-environment mate-desktop-environment-extra -y
```

### 日本語入力


　日本語のIMEは[Mozc](https://github.com/google/mozc)を使います。

```
$ sudo apt-get install ibus-mozc -y
```


 `~/.xsession`にibusとMATEの起動を設定します。

```
$ cat <<EOF > ~/.xsession
export GTK_IM_MODULE=ibus
export QT_IM_MODULE=ibus
export XMODIFIERS="@im=ibus"
ibus-daemon -rdx
mate-session
EOF
```

 rebootするとxrdp接続でMATEのリモートデスクトップ環境が使えるようになります。

```
$ sudo reboot
```


　IDCFクラウドの管理コンソースからIPアドレスを追加して、3389ポートのファイアウォールと作成した仮想マシンへのポートフォワードを設定します。Windows 10の場合標準のリモートデスクトップが使えます。追加したIPアドレスを指定して接続します。macOSの場合は日本語キーボードが使える[Microsoft Remote Desktop Connection Client for Mac 2.1.1](http://www.microsoft.com/ja-jp/download/details.aspx?id=18140)をインストールします。

　管理ユーザーとして作成した`cloud-user`とパスワードを入力してログインします。


![ubuntu-mate.png](/2017/02/04/idcf-ubuntu-1610-xrdp/ubuntu-mate.png)


　MATE Terminalを開きます。

* Applications -> Sytem Tools -> MATE Terminal

　`ibus-setup`を実行して日本語IMEにMozcを指定します。

```
$ ibus-setup
```

 * Input Method -> Add -> Japanese -> Mozc -> Add


 Fireroxを使い日本語入力を確認します。

```
$ sudo apt-get install -y firefox
```

* Applications -> Internet -> Firefox Web Browser


　右上メニューをJapanese-Japanese から Japanese-Mozcに変更するとアイコンが「あ」になり日本語入力ができるようになります。


![ubuntu-mate-japanese.png](/2017/02/04/idcf-ubuntu-1610-xrdp/ubuntu-mate-japanese.png)


　Windows 10の場合はの半角/全角キー、macOSの場合は半角キーで日本語入力を切り換えることができます。

