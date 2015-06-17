title: "Intel Edisonの初期設定とChromebookからシリアル接続してWi-Fiの確認をする"
date: 2015-03-06 11:43:23
tags:
 - Chromebook
 - IntelEdison
description: EdisonからChromebookのUSBテザリングを使いインターネットに接続することができました。ファームウェアの更新とEdisonの初期設定をして開発の準備をします。EdisonはNode.jsがデフォルトでインストールされているので初期設定が終了したら開発に入ることができます。
---

EdisonからChromebookの[USBテザリング](/2015/03/05/intel-edison-chromebook-usb-tethering/)を使いインターネットに接続することができました。ファームウェアの更新とEdisonの初期設定をして開発の準備をします。EdisonはNode.jsがデフォルトでインストールされているので初期設定が終了したら開発に入ることができます。

<!-- more -->

## ファームウェアの更新

工場出荷状態のファームウェアは古いようなので最新のイメージをダウンロードして更新します。

### Chromebookでイメージのダウンロード

ChromebookとEdisonのJ16のUSBコネクタを接続するとファイルアプリからEdisonの名前でドライブがマウントされます。まだフォルダには何もはいっていません。

![edison-mount.png](/2015/03/06/intel-edison-setting-up/edison-moung.png)

[Intel Edison Software Download](http://www.intel.com/support/edison/sb/CS-035180.htm)サイトから[Yocto complete image](http://downloadmirror.intel.com/24698/eng/edison-image-ww05-15.zip)をダウンロードします。ダウンロードした`edison-image-ww05-15.zip`をダブルクリックして展開して、中身をすべてEdisonフォルダにコピーします。

### EdisonにSSH接続

Chromeブラウザを開き、`ctrl + alt + t`からcroshを起動してshellを実行します。

``` bash
crosh> shell
```

EdisonはJ16コネクタをChromebookと接続してUSBテザリングします。Edisonを使いたいときにネットワーク設定をするスクリプトを用意します。

``` bash ~/Downloads/edison-setup.sh
sudo sh -c 'echo 1 >> /proc/sys/net/ipv4/ip_forward'
sudo ifconfig usb0 192.168.2.1 netmask 255.255.255.0
sudo iptables -t nat -A POSTROUTING -o mlan0 -j MASQUERADE
sudo iptables -A FORWARD -i mlan0 -o usb0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i usb0 -o mlan0 -j ACCEPT
ssh root@192.168.2.15
```

スクリプトを実行してEdisonにSSH接続します。

``` bash
$ bash ~/Downloads/edison-setup.sh
```

デフォルトゲートウェイの設定をしてインターネット接続ができることを確認します。

``` bash
$ route add default gw 192.168.2.1
$ route -n
$ ping -c 1 www.yahoo.co.jp
```

### Edisonでファームウェアの更新

ファームウェアの更新前のバージョンは`edison-weekly_build_56_2014-08-20_15-54-05`です。

``` bash
$ cat /etc/version 
edison-weekly_build_56_2014-08-20_15-54-05
```

`reboot ota`コマンドを実行するとEdisonが再起動します。ファームウェアが更新されるのでしばらく待ちます。

```
$ reboot ota
```

Chromebookからリムーバルデバイスが再認識されるのでEdisonにSSH接続します。更新されたバージョンを確認すると`weekly-120`でした。`configure_edison`コマンドを`--upgrade`フラグを付けて実行すると、ファームウェアは最新版になったようです。今後はコマンドからファームウェアの更新ができるようになります。

``` bash
$ cat /etc/version 
weekly-120
$ configure_edison --upgrade
The latest version is already installed.
```

## 初期設定

`configure_edison`コマンドを`--setup`フラグを付けて実行して初期設定をします。

``` bash
$ configure_edison --setup
```

### パスワードとデバイス名

ダイアログでは最初にパスワード設定をします。

``` bash
Configure Edison: Device Password

Enter a new password (leave empty to abort)
This will be used to connect to the access point and login to the device.
Password:       *********
Please enter the password again:        *********
```

次にデバイス名を入力します。edisonにしました。

``` bash
Configure Edison: Device Name

Give this Edison a unique name.
This will be used for the access point SSID and mDNS address.
Make it at least five characters long (leave empty to skip): edison
Is edison correct? [Y or N]: Y

Do you want to set up wifi? [Y or N]: Y
```

### Wi-Fiの設定

次にWi-Fiの設定に入ります。今回の環境ではWi-Fiのアクセスポイントはステルスモードなので、`2 (Manually input a hidden SSID)`をタイプしてSSIDを手動で入力します。暗号化方式は環境にあわせて設定します。

``` bash
Configure Edison: WiFi Connection

Scanning: 1 seconds left  

0 :     Rescan for networks
1 :     Exit WiFi Setup
2 :     Manually input a hidden SSID
....

Enter 0 to rescan for networks.
Enter 1 to exit.
Enter 2 to input a hidden network SSID.

Enter a number between 3 to 6 to choose one of the listed network SSIDs: 2
Please enter the hidden network SSID: xxxx
Is xxxx correct? [Y or N]: Y

    0: OPEN
    1: WEP
    2: WPA-Personal(PSK)
    3: WPA-Enterprise (EAP)
  
Select the type of security [0 to 3]: 2
Password must be between 8 and 63 characters.
What is the network password?: 
Initiating connection to xxxx. Please wait...
Attempting to enable network access, please check 'wpa_cli status' after a minute to confirm.
Done. Please connect your laptop or PC to the same network as this device and go to http://xxx.xxx.xxx.xxx or http://edison.local in your browser.
```

ダイアログで入力したWi-Fiの設定はRaspberry Piと同様に`wpa_supplicant.conf`に保存されます。`scan_ssid=1`も入っています。

``` bash /etc/wpa_supplicant/wpa_supplicant.conf
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=0
config_methods=virtual_push_button virtual_display push_button keypad
update_config=1
fast_reauth=1
device_name=Edison
manufacturer=Intel
model_name=Edison

network={
  ssid="xxx"
  scan_ssid=1
  key_mgmt=WPA-PSK
  pairwise=CCMP TKIP
  group=CCMP TKIP WEP104 WEP40
  eap=TTLS PEAP TLS
  psk="xxx"
}
```

## パッケージの更新

opkgコマンドを使いパッケージの更新をします。まず`opkg update`してパッケージのリストを更新します。

``` bash
$ opkg update
Downloading http://iotdk.intel.com/repos/1.1/intelgalactic/Packages.
Updated list of available packages in /var/lib/opkg/iotkit.
```

`opkg upgrade`を実行するとlibmraa0とupmパッケージが更新されました。

``` bash
$ opkg upgrade
...
upm (0.1.9) already install on root.
libmraa0 (0.6.1) already install on root.
Configuring libmraa0.
Configuring upm.
```

## 時刻合わせ

timedatectlコマンドを実行するとタイムゾーンがUTCになっています。

``` bash
$ timedatectl
      Local time: Fri 2015-03-06 01:33:41 UTC
  Universal time: Fri 2015-03-06 01:33:41 UTC
        RTC time: Fri 2015-03-06 01:33:41
       Time zone: Universal (UTC, +0000)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: n/a
```

タイムゾーンを`Asia/Tokyo`に設定します。

``` bash
$ timedatectl set-timezone Asia/Tokyo
$ timedatectl
      Local time: Fri 2015-03-06 10:34:59 JST
  Universal time: Fri 2015-03-06 01:34:59 UTC
        RTC time: Fri 2015-03-06 01:34:59
       Time zone: Asia/Tokyo (JST, +0900)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: n/a
```

## Node.js

EdisonはデフォルトでNode.jsとnpmがデフォルトで使えます。

``` bash
$ node -v
v0.10.28
$ npm -v
1.4.9
```

## Chromebookからシリアル接続をしてWi-Fiの確認

ChromebookとJ16コネクタで接続していましたが、シリアル接続の場合はJ3コネクタとつなぎます。J16コネクタは電源アダプタに接続します。Chromeウェブストアから[Serial Monitor](https://chrome.google.com/webstore/detail/serial-monitor/ohncdkkhephpakbbecnkclhjkmbjnmlo)をインストールしてシリアル接続のテストをします。

![port-monitor.png](/2015/03/06/intel-edison-setting-up/port-monitor.png)

pingコマンドを送信して、インターネット接続の確認ができました。

![ping-test.png](/2015/03/06/intel-edison-setting-up/ping-test.png)
