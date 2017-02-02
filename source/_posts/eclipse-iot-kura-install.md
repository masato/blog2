title: "Eclipse IoT の紹介 - Part1: Eclipse KuraをRaspberry Piにインストールする"
date: 2017-01-29 15:48:28
categories:
 - IoT
 - EclipseIoT
tags:
 - EclipseIoT
 - EclipseKura
description: IoT Gatewayについて調査をしていくつかのプロダクトを試しました。Eclipse Kuraが今のところ実績もあり使いやすそうです。Eclipse IoTプロジェクトの一つで、Raspberry PiやBeagle Bone Black上でIoT Gatewayとして動作するOSGiのオープンソースフレームワークです。
---


　[IoT Gatewayについて](http://qiita.com/masato/items/912a447698f172dbb45b)調査をしていくつかのプロダクトを試しました。[Eclipse Kura](http://www.eclipse.org/kura/)が今のところ実績もあり使いやすそうです。[Eclipse IoTプロジェクト](http://iot.eclipse.org/)の一つで、Raspberry PiやBeagle Bone Black上でIoT Gatewayとして動作する[OSGi](https://www.osgi.org/)のオープンソースフレームワークです。Eclipse Kuraを拡張した[ESF](http://www.eurotech.com/jp/products/software+services/everyware+software+framework)とハードウェアの[ReliaGATE](http://www.eurotech.com/jp/products/software+services/m2m+products)シリーズも[Eurotech](http://www.eurotech.com/)から提供されています。プロダクション環境でサポートが必要な場合はライセンスを購入すると安心できそうです。

<!-- more -->

　[Eclipse IoT](https://iot.eclipse.org/)プロジェクトは他にも[Eclipse Kapua](https://projects.eclipse.org/proposals/eclipse-kapua)、[Eclipse Hono](https://projects.eclipse.org/projects/iot.hono)、[Eclipse Vorto](http://www.eclipse.org/vorto/)、[Eclipse Leshan](http://www.eclipse.org/leshan/)といったクラウドのバックエンド側プロジェクトもあります。[Red Hat](https://www.redhat.com/en)、[Eurotech](http://www.eurotech.com/en/)、[Bosh](http://www.bosch.com/en/com/home/index.php)、[GE Digital](https://www.ge.com/digital/)といった企業が乱立気味のIoTプラットフォームの相互接続性と、デバイスからのデータをエンタープライズのアプリケーションに統合することを目標にオープンソースで開発を進めています。バックエンドサービスは[Cloud Foundry](https://www.cloudfoundry.org/)や[OpanShift](https://www.openshift.com/)といった最近のDockerベースのクラウドプラットフォーム上で動作することにも注目です。

　まずは身近なところから、Raspberry Pi 2にEclipse Kuraをインストールすることから始めます。

## Raspbian Jessie Lite

　Raspbianのセットアップ方法は[オフィシャル](https://www.raspberrypi.org/documentation/installation/installing-images/)をはじめにたくさん記事がありますが、2016-11-25の[リリース]( http://downloads.raspberrypi.org/raspbian/release_notes.txt
)からSSHがデフォルトで無効になり注意が必要です。SDカードにイメージを焼いた状態のままではSSHで接続することができません。


>2016-11-25:
  * SSH disabled by default; can be enabled by creating a file with name "ssh" in boot partition


　ローカルの作業用PCとRaspberry Pi 2を有線LANで接続してヘッドレスインストールする例を簡単にまとめておきます。そのほかには無線LAN USBアダプターなどが必要です。


* [無線LAN USBアダプター](https://www.amazon.co.jp/dp/B00ESA34GA/)

* [有線LAN USBアダプター](https://www.amazon.co.jp/dp/B00LVH885U/)

* [LANケーブル](https://www.amazon.co.jp/dp/B008RVY4GK/)

### SDカードにイメージを焼く

　SDカードをUSBアダプタに挿して作業用PCに接続します。Fat32でフォーマットされたSDカードのデバイス名を確認してアンマウントします。この例では/dev/disk2です。

```
$ diskutil list
$ diskutil unmountDisk /dev/disk2
```

　Raspbian Jessie Liteを[ダウンロードページ](https://www.raspberrypi.org/downloads/raspbian/)から取得してイメージを解凍します。

```
$ cd ~/Downloads
$ wget https://downloads.raspberrypi.org/raspbian/images/raspbian-2017-01-10/2017-01-11-raspbian-jessie.zip
$ unzip 2017-01-11-raspbian-jessie.zip
```

　イメージをSDカードに焼きます。確認したデバイス名に`r`をつけて`/dev/rdisk2`を指定します。

```t
$ sudo dd bs=1m if=2017-01-11-raspbian-jessie-lite.img of=/dev/rdisk2
```

　Windows 10でRasbianイメージのSDカードを作る場合は[Win32 Disk Imager](https://ja.osdn.net/projects/sfnet_win32diskimager/)が便利です。

### SSHを有効にする

　SSHはデフォルトで無効になっています。macOSのボリュームにマウントした状態でsshファイルを作り有効にします。

```
$ touch /Volumes/boot/ssh
```

　Windows 10には`touch`に相当するコマンドがないため、SDカードのDドライブに移動して`COPY`コマンドで代用します。エクスプローラーからDドライブに直接ファイルを作成する場合は`.txt`など拡張子をつけないように注意します。

```
> d:
> copy /y nul ssh
> exit
```

　SDカードをアンマウントして取り出します。

```
$ diskutil unmountDisk /dev/disk2
```

　SDカードをRaspberry Pi 2にさして電源を入れます。macOSとLANケーブルで直接つなぎます。最新のRaspbianはデフォルトでmDNSが有効になりmacOSやWindows10から`raspberrypi.local`のホスト名で簡単に接続できます。

```
$ ssh pi@raspberrypi.local
```

　ユーザー名は`pi`、パスワードは`raspbian`が設定されています。SSHのデフォルトが無効なのは変更せずにこのまま使う人が多いためらしく、忘れずにパスワードを変更します。

```
$ passwd
```

　公開鍵を`~/.ssh/authorized_keys`に追加しSSH接続で利用します。

```
$ mkdir ~/.ssh
$ chmod 700 ~/.ssh
$ cat <<EOF > ~/.ssh/authorized_keys
ssh-rsa AAAAB...
EOF
$ chmod 600 ~/.ssh/authorized_keys
$ exit
```

### 初期設定

　raspi-configから初期設定を行います。

```
$ sudo raspi-config
```

　ファイルシステムの拡張は必ず行います。localeとtimezoneはユーザー環境にあわせて設定します。

* 1 Expand Filesystem
* 4 Internationalisation Options
  * I1 Change Locale -> en_GB.UTF-8のまま
  * I2 Change Timezone -> Asia -> Tokyo

　localeを`jp_JP.UTF-8`にすると文字化けする場合があるので`en_GB.UTF-8`のまま使います。一度rebootします。


　Raspberry Piに日本語キーボードを使う場合はレイアウトを変更します。SSH接続では不要ですが、Windows 10とRaspberry Piを有線LANで接続するとmDNSの`raspberrypi.local`でうまくSSH接続できないことがあります。キーボードとディスプレイが必要になる場合があるため設定します。

 * 4 Localisation Options
  * I3 Change Keyboard Layout  -> Generic 105-key (Intl) PC -> Japanese ->  The default for the keyboard layout ->  No compose key

```
$ sudo reboot
```

### 無線LAN

　Raspberry Pi 2には無線LANが内蔵されていません。USBの無線LANアダプタを用意しておきます。wlan0で利用する無線LANのアクセスポイントをスキャンしてESSIDを確認します。

```
$ sudo iwlist wlan0 scan | grep ESSID
```

　wpa_passphraseコマンドにESSIDとパスワードを渡し、出力を`wpa_supplicant.conf`に追記します。(例の`[`と`]`は不要です。)

```
$ sudo sh -c 'wpa_passphrase [ESSID] [パスワード] >> /etc/wpa_supplicant/wpa_supplicant.conf'
```

　アクセスポイントを複数指定する場合は`priority`で接続する優先度を指定します。数字が大きいほど有線されます。

```/etc/wpa_supplicant/wpa_supplicant.conf
country=GB
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
network={
        ssid="xxx"
        #psk="xxx"
        psk=xxx
        priority=0
}
network={
        ssid="xxx"
        #psk="xxx"
        psk=xxx
        priority=1
}
```

　wlan0のインタフェースにDHCPと先ほどのwpa_supplicant.confを設定します。

```bash:/etc/network/interfaces
auto lo
iface lo inet loopback

iface eth0 inet manual

allow-hotplug wlan0
iface wlan0 inet dhcp
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```

　wlan0を再起動してDHCPでIPアドレスを取得します。

```
$ sudo ifdown wlan0
$ sudo ifup wlan0
```

 `ip`コマンドなどを使いwlan0が正常に起動しているか確認します。

```
$ sudo iwconfig wlan0
$ ip addr show wlan0
$ ping -c 1 www.yahoo.co.jp
```

## パッケージの更新

　ネットワークがつながるようになったので以降の作業を進める前にインストールされているパッケージを更新します。

### apt-get ミラーサイト

　apt-getのデフォルトは`mirrordirector.raspbian.org`のリポジトリに接続します。

```/etc/apt/sources.list
deb http://mirrordirector.raspbian.org/raspbian/ jessie main contrib non-free rpi
# Uncomment line below then 'apt-get update' to enable 'apt-get source'
#deb-src http://archive.raspbian.org/raspbian/ jessie main contrib non-free rpi
```

　なるべく近くのミラーサイトを[RaspbianMirrors](http://www.raspbian.org/RaspbianMirrors)から選びます。今回は[JAIST](http://ftp.jaist.ac.jp/raspbian)にしました。

```
$ sudo sed -i".bak" -e "s/mirrordirector.raspbian.org/ftp.jaist.ac.jp/g" /etc/apt/sources.list
```

### パッケージ更新

 パッケージリストを更新してパッケージを最新の状態にします。

```
$ sudo apt-get update && sudo apt-get dist-upgrade -y
```

　nanoエディタが使いづらい場合はvimをインストールしてファイルを編集します。

```
$ sudo apt-get update && sudo apt-get install vim -y
```

## Eclipse Kuraのインストール

　Raspbianの初期設定が終わったところでようやくEclipse Kuraのインストールを始めていきます。

### OpenJDK 8

　Eclipse Kuraの動作には[JDK 7以上](https://wiki.eclipse.org/Kura/Getting_Started#Java)が必要です。最新の[Open JDK 8](http://openjdk.java.net/)を使います。

```
$ sudo apt-get update && sudo apt-get install -y openjdk-8-jre-headless
$ java -version
openjdk version "1.8.0_40-internal"
OpenJDK Runtime Environment (build 1.8.0_40-internal-b04)
OpenJDK Zero VM (build 25.40-b08, interpreted mode)
```

### Eclipse Kura 2.0.2

　[Raspberry Pi Quick Start](https://eclipse.github.io/kura/doc/raspberry-pi-quick-start.html)の手順に従いKuraをインストールします。

　dhcpcd5 パッケージはKuraと互換性がないため削除します。

```
$ sudo apt-get purge -y dhcpcd5
```

　NetworkManagerもKuraと競合するためインストールされていないことを確認します。

```
$ sudo apt-get remove -y network-manager
```

　パッケージの依存関係を解決してKuraをインストールするために[GDebi](https://launchpad.net/gdebi
)をインストールします。

```
$ sudo apt-get update && sudo apt-get install -y gdebi-core
```

　[Eclipse Kura 2.0.2](https://projects.eclipse.org/projects/iot.kura/releases/2.0.2)をダウンロードして使います。

```
$ wget http://eclipse.stu.edu.tw/kura/releases/2.0.2/kura_2.0.2_raspberry-pi-2_installer.deb
$ sudo gdebi kura_2.0.2_raspberry-pi-2_installer.deb
```

　rebootするとEclipse Kuraが起動します。

```
$ sudo reboot
```

　Eclipse Kuraの起動には時間がかかるためSSH接続したらログを

```
$ sudo tail /var/log/kura.log
```

## Eclipse Kura WebUI

　Raspberry PiにWebブラウザから以下のURLに接続するとWebコンソールが表示されます。

Eclipse Kura WebUI
http://raspberrypi.local/kura

　Basic認証のユーザー名とパスワードはadminです。

* username: admin
* password: admin

　Cloud ServiceやNetwork設定はこれからですが、とりあえずEclipse Kuraのインストールは終了です。

![kapua-console.png](/2017/01/29/eclipse-iot-kura-install/kapua-console.png)
