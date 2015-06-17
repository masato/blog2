title: 'Salt チュートリアル - Part4: モジュール'
date: 2014-09-11 20:04:14
tags:
---


### test,cmdモジュール

testモジュールのping関数を使い、saltの通信テストをします。

``` bash
$ salt 'minion*' test.ping
minion1.cs29dcloud.internal:
    True
minion2.cs29dcloud.internal:
    True
```

cmdモジュールのrun関数で、minionにコマンドを実行します。
`/var/log`ディレクトリをlsしてみます。
`salt.cs29dcloud.internal`は、masterのサーバーにインストールしたminionです。

``` bash
$ salt "*" cmd.run "ls /var/log -altr"
salt.cs29dcloud.internal:
    total 704
    drwxr-xr-x  2 ntp    ntp    4096 Jun  6  2012 ntpstats
    drwxr-xr-x  2 root   root   4096 Oct 11  2012 dist-upgrade
    drwxr-xr-x  2 root   root   4096 Nov 15  2012 unattended-upgrades
    drwxr-xr-x  2 root   root   4096 Apr 11 19:08 fsck
    drwxr-xr-x  2 root   root   4096 Apr 11 19:08 installer
    -rw-r--r--  1 root   root      0 Apr 11 19:08 faillog
    -rw-rw----  1 root   utmp      0 Apr 11 19:08 btmp
...
```

