title: "Chromebookにdev_installしてScreenでEdisonにシリアル接続する"
date: 2015-03-07 19:02:04
tags:
 - IntelEdison
 - Chromebook
 - Screen
 - dev_install
 - Gentoo
description: Chromebookからは普通はscreenコマンドが使えないのでEdisonやRaspberry PiにUSBシリアル接続をするのが難しいです。ググっているとdev_installをするとscreenが使えるようです。GentooベースのChrome OSではパッケージ管理にPortageが採用されています。emergeコマンドを使ってパッケージをインストールすることができます。
---

Chromebookからは普通はscreenコマンドが使えないのでEdisonやRaspberry PiにUSBシリアル接続をするのが難しいです。ググっているとdev_installをするとscreenが使えるようです。GentooベースのChrome OSではパッケージ管理にPortageが採用されています。emergeコマンドを使ってパッケージをインストールすることができます。

<!-- more -->

## dev_install

`cntl-alt-t`でcroshを起動してから、shellでrootになります。

``` bash
crosh> shell
$ sudo -i
```

`dev_install`コマンドを実行します。

``` bash
$ dev_install
Starting installation of developer packages.
First, we download the necessary files.
Downloading https://commondatastorage.googleapis.com/chromeos-dev-installer/board/daisy_spring/canary-R40-6457.107.0/packages/sys-apps/sandbox-2.6-r1.tbz2
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  252k  100  252k    0     0   274k      0 --:--:-- --:--:-- --:--:--  276k
Extracting /usr/local/portage/packages/sys-apps/sandbox-2.6-r1.tbz2
...
Extracting /usr/local/portage/packages/net-misc/dhcp-4.2.2-r2.tbz2
Files downloaded, configuring emerge.
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
curl: (22) The requested URL returned error: 404 Not Found
Emerge installation complete. Installing additional optional packages.
######################################################################## 100.0%
!!! PORTAGE_BINHOST unset, but use is requested.
emerge: incomplete set configuration, missing set(s): "selected", "system", and "world"
        sets defined: x11-module-rebuild
        This usually means that '/usr/share/portage/config/sets/portage.conf'
        is missing or corrupt.
        Falling back to default world and system set configuration!!!
!!! Problem with sandbox binary. Disabling...

Calculating dependencies... done!

>>> Emerging binary (1 of 1) chromeos-base/gmerge-0.0.1-r837::chromiumos for /usr/local/
######################################################################## 100.0%
 * gmerge-0.0.1-r837.tbz2 MD5 SHA1 size ;-) ...                          [ ok ]
>>> Extracting info
 * Running stacked hooks for pre_pkg_setup
 *    sysroot_build_bin_dir ...                                                                                 [ ok ]
>>> Extracting chromeos-base/gmerge-0.0.1-r837

>>> Installing (1 of 1) chromeos-base/gmerge-0.0.1-r837::chromiumos to /usr/local/
 * Running stacked hooks for pre_pkg_preinst
 *    wrap_old_config_scripts ...                                                                               [ ok ]

>>> Recording chromeos-base/gmerge in "world" favorites file...
>>> Auto-cleaning packages...

>>> Using system located in ROOT tree /usr/local/

>>> No outdated packages were found on your system.
Install virtual/target-os-dev package now? (y/N) 
Local copy of remote index is up-to-date and will be used.
!!! PORTAGE_BINHOST unset, but use is requested.
emerge: incomplete set configuration, missing set(s): "selected", "system", and "world"
        sets defined: x11-module-rebuild
        This usually means that '/usr/share/portage/config/sets/portage.conf'
        is missing or corrupt.
        Falling back to default world and system set configuration!!!
!!! Problem with sandbox binary. Disabling...

Calculating dependencies... done!

>>> Emerging binary (1 of 89) net-dialup/lrzsz-0.12.20-r3::portage-stable for /usr/local/
 * Fetching '/usr/local/portage/packages/net-
 * dialup/lrzsz-0.12.20-r3.tbz2' in the background. To view fetch
 * progress, run `tail -f /var/log/emerge-fetch.log` in another
 * terminal.
 * lrzsz-0.12.20-r3.tbz2 MD5 SHA1 size ;-) ...                           [ ok ]
```

たくさんエラーが出てインストールに失敗したように思いますが、[dev_install doesn't work on Samsung Arm Chromebook 2 (Peach Pi)](https://code.google.com/p/chromium/issues/detail?id=410195)によると、以下の3つは重大なエラーではないそうです。解かりづらいですが。

* !!! PORTAGE_BINHOST unset, but use is requested.
* emerge: incomplete set configuration, missing set(s): "selected", "system", and "world"
* !!! Problem with sandbox binary. Disabling...


バージョンを確認しても警告がでます。

``` bash
$ emerge --version
!!! No gcc found. You probably need to 'source /etc/profile'
!!! to update the environment of this terminal and possibly
!!! other terminals also.
Portage 2.2.12-r6 (python 2.7.3-final-0, unavailable, [unavailable], unavailable, 3.8.11 armv7l)
```

## QEmacsをインストール

QEmacsをインストールします。

``` bash
$ emerge qemacs
...
>>> Recording app-editors/qemacs in "world" favorites file...

 * Messages for package app-editors/qemacs-0.4.0_pre20090420 merged to /usr/local/:

 * One or more symlinks to directories have been preserved in order to
 * ensure that files installed via these symlinks remain accessible. This
 * indicates that the mentioned symlink(s) may be obsolete remnants of an
 * old install, and it may be appropriate to replace a given symlink with
 * the directory that it points to.
 * 
 *      /usr/local/usr
 * 
>>> Auto-cleaning packages...

>>> Using system located in ROOT tree /usr/local/

>>> No outdated packages were found on your system.
```

バージョンを確認してみます。

``` bash
$ which qemacs
/usr/local/bin/qemacs
$ qemacs --version
QEmacs version 0.4.0dev
Copyright (c) 2000-2003 Fabrice Bellard
Copyright (c) 2000-2008 Charlie Gordon

QEmacs comes with ABSOLUTELY NO WARRANTY.
You may redistribute copies of QEmacs
under the terms of the GNU Lesser General Public License.
```

qemacsを起動します。

``` bash
$ qemacs
```

起動に成功しました。日本語は入力できませんがEmacsは使うことができます。

![qemacs.png](/2015/03/07/chromebook-dev-install-screen-edison/qemacs.png)


## Screen

Screenを使う前に、Chromebookでは`/root`などはRead-Onlyなのでファイルが書き込めるようにします。sudoでscreenを使いたいので`/root/.screen`ディレクトリを作成できるようにします。

``` bash
$ sudo /usr/share/vboot/bin/make_dev_ssd.sh --remove_rootfs_verification --partitions 4
Kernel B: Disabled rootfs verification.
Backup of Kernel B is stored in: /mnt/stateful_partition/backups/kernel_B_20150308_184934.bin
Kernel B: Re-signed with developer keys successfully.
Successfully re-signed 1 of 1 kernel(s)  on device /dev/mmcblk0.
```

設定後は一度再起動が必要です。

``` bash
$ sudo reboot
```

## EdisonとUSB-Serial接続

EdisonのJ3コネクタとChromebookをUSB-Serial接続することができました。

``` bash
$ sudo screen /dev/ttyUSB0 115200
Poky (Yocto Project Reference Distro) 1.6.1 edison ttyMFD2

edison login: root
Password: 
root@edison:~# 
```
