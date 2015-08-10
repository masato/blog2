title: "Raspberry Piをngrokで公開する - Part5: Webcamでアボカドをライブ配信する"
date: 2015-08-08 16:23:06
categories:
 - IoT
tags:
 - RaspberryPi
 - Webcam
 - Supervisor
 - Motion
 - アボカド
 - freeboard
 - C270
description: 水耕栽培しているアボカドの植物育成LEDライトの点灯や消灯をHubotを使って実装しています。リモートからライトを制御できるのですが、様子が見えないので本当にライトが点いたどうか分かりません。Raspberry PiにWebcamをつけてアボカドをライブ配信しながらライトの制御を確認してみます。
---

水耕栽培しているアボカドの[植物育成LEDライトの点灯や消灯](/2015/07/15/iot-avocado-growth-monitoring-mqtt-raspi-bbb/)を[Hubot](https://hubot.github.com/)を使って実装しています。リモートからライトを制御できるのですが、様子が見えないので本当にライトが点いたどうか分かりません。Raspberry PiにWebcamをつけてアボカドをライブ配信しながらライトの制御を確認してみます。

<!-- more -->

## Logicool C270

Webcamは安価な[Logicool C270](http://www.logicool.co.jp/ja-jp/product/hd-webcam-c270?crid=34)を2,000円くらいで購入しました。国内外でRaspberry Piと一緒に使う例がたくさん見つかるので安心です。

* [Raspberry PiのWebカメラをLogicool C270にグレードアップする](http://denshikousaku.net/logicool-c270-raspberry-pi)
* [How to make a DIY home alarm system with a raspberry pi and a webcam](https://medium.com/@Cvrsor/how-to-make-a-diy-home-alarm-system-with-a-raspberry-pi-and-a-webcam-2d5a2d61da3d)


## Motion

### Raspberry Piの動画配信ツール

Raspberry PiにWebcamをUSBで接続して動画配信をするツールを調べるといくつか見つかりました。

* [Motion](http://www.lavrsen.dk/foswiki/bin/view/Motion/WebHome)
* [UV4L](http://www.linux-projects.org/modules/sections/index.php?op=viewarticle&artid=14)
* [MJPG-streamer](http://sourceforge.net/projects/mjpg-streamer/)
* [Socket.IO binary support](http://yosuke-furukawa.hatenablog.com/entry/2014/06/25/111615)
* [WebSocket](http://ami-gs.hatenablog.com/entry/2014/04/09/230224)

今回はdebパッケージでインストールできて、設定が簡単な[Motion](http://www.lavrsen.dk/foswiki/bin/view/Motion/WebHome)を使ってみます。


### Motionのインストール

Raspberry PiにWebcamをUSB接続します。lsusbコマンドを実行してカメラが認識されているか確認します。

```bash
$ lsusb
...
Bus 001 Device 005: ID 046d:0825 Logitech, Inc. Webcam C270
```

Motionをapt-get installします。`motion`ユーザーと`video`グループも同時に作成されました。

```bash
$ sudo apt-get update
$ sudo apt-get install motion
...
Setting up motion (3.2.12-3.4) ...
Adding group `motion' (GID 111) ...
Done.
Adding system user `motion' (UID 108) ...
Adding new user `motion' (UID 108) with group `motion' ...
Not creating home directory `/home/motion'.
Adding user `motion' to group `video' ...
Adding user motion to group video
Done.
Not starting motion daemon, disabled via /etc/default/motion ... (warning).
Setting up ffmpeg (6:0.8.17-1+rpi1) ...
```

### 設定ファイル


最初に設定ファイルのバックアップをとります。

```bash
$ sudo cp /etc/motion/motion.conf{,.orig}
```

motion.confを以下のように修正します。設定は以下のサイトを参考にさせていただきました。

* [motionでお手軽監視カメラをつくる](http://blog.hello-world.jp.net/raspberrypi/1949/)
* [Motion でライブカメラ+動体検知サーバー構築](http://debianj.com/ubuntu/webcam/motion)

```bash /etc/motion/motion.conf
width 640             # 幅
height 480            # 高さ
framerate 30          # フレームレート、1秒間に撮影する枚数
output_normal off     # 画像ファイルを保存しない
webcam_localhost off  # ローカルホスト以外もライブ映像を配信
```

## Motionの起動

### ノンデーモン起動

テスト用にMotionをnon-daemonモードで起動します。

```bash
$ sudo motion -n
```

Raspberry Piはローカルネットワーク上にあります。

http://192.168.128.203:8081/

同じLANにOSXを接続してブラウザを開きます。Chromeは直接URLから開けませんでした。

* Chrome: x
* Safari: ○
* Firefox: ○

### デーモン起動

Motionをデモナイズする場合、設定ファイルは2つ修正が必要です。`/etc/default/motion`の`start_motion_daemon`をyesにします。

```bash /etc/default/motion
# set to 'yes' to enable the motion daemon
#start_motion_daemon=no
start_motion_daemon=yes
```

`/etc/motion/motion.conf`の`daemon`をonにします。

```bash /etc/motion/motion.conf
# Start in daemon (background) mode and release terminal (default: off)
#daemon off
daemon on
```

`/etc/init.d/motion`が有効になるのでMotionのサービスを起動します。

```bash
$ sudo service motion start
Starting motion detection daemon: motion.
```


## 設定ファイルのdiff

設定ファイルの変更箇所をdiffで確認します。

```bash
$ sudo diff -u motion.conf.orig motion.conf > /tmp/motion.diff
```

```text /tmp/motion.diff
--- motion.conf.orig    2015-08-04 15:32:52.354807944 +0900
+++ motion.conf 2015-08-05 12:56:32.980973204 +0900
@@ -8,7 +8,8 @@
 ############################################################

 # Start in daemon (background) mode and release terminal (default: off)
-daemon off
+#daemon off
+daemon on

 # File to store the process ID, also called pid file. (default: not defined)
 process_id_file /var/run/motion/motion.pid
@@ -67,14 +68,17 @@
 rotate 0

 # Image width (pixels). Valid range: Camera dependent, default: 352
-width 320
+#width 320
+width 640

 # Image height (pixels). Valid range: Camera dependent, default: 288
-height 240
+#height 240
+height 480

 # Maximum number of frames to be captured per second.
 # Valid range: 2-100. Default: 100 (almost no limit).
-framerate 2
+#framerate 2
+framerate 30

 # Minimum time in seconds between capturing picture frames from the camera.
 # Default: 0 = disabled - the capture rate is given by the camera framerate.
@@ -224,7 +228,8 @@
 # Picture with most motion of an event is saved when set to 'best'.
 # Picture with motion nearest center of picture is saved when set to 'center'.
 # Can be used as preview shot for the corresponding movie.
-output_normal on
+#output_normal on
+output_normal off

 # Output pictures with only the pixels moving object (ghost images) (default: off)
 output_motion off
@@ -410,7 +415,8 @@
 webcam_maxrate 1

 # Restrict webcam connections to localhost only (default: on)
-webcam_localhost on
+#webcam_localhost on
+webcam_localhost off

 # Limits the number of images per connection (default: 0 = unlimited)
 # Number can be defined by multiplying actual webcam rate by desired number of seconds
```

## ngrok

ngrokの[ダウンロード](https://ngrok.com/download)ページからLinux/ARMのバイナリをRaspberry Piにインストールします。

```bash
$ wget https://dl.ngrok.com/ngrok_2.0.19_linux_arm.zip
$ sudo unzip ngrok_2.0.19_linux_arm.zip -d /usr/local/bin
Archive:  ngrok_2.0.19_linux_arm.zip
  inflating: /usr/local/bin/ngrok
$ ngrok version
ngrok version 2.0.19
```

最初にngrokをフォアグラウンドで起動してテストします。MotionのHTTP
の8081ポートをngrokを使ってトンネルします。

```bash
$ ngrok http 8081
...
Forwarding                    http://0a08cc8b.ngrok.io -> localhost:8081
Forwarding                    https://0a08cc8b.ngrok.io -> localhost:8081
```

SafariかFirefoxで開いてWebcamの動画が表示されていることを確認します。

![avocado-webcam.png](/2015/08/08/raspberrypi-ngrok-motion/avocado-webcam.png)

### Supervisor

[Supervisor](http://supervisord.org/)を使いngrokのトンネルのプロセスを管理します。設定ファイルの`command`行にngrokコマンドを記述します。`--authtoken`フラグと`-subdomain`フラグを追加します。`-subdomain`フラグで指定するサブドメイン名は[Reserved Tunnels](https://dashboard.ngrok.com/reserved)画面のReserved Domainセクションから登録しておきます。

```bash /etc/supervisor/conf.d/ngrok-webcam.conf
[program:ngrok-webcam]
command=/usr/local/bin/ngrok http --authtoken xxx -subdomain=xxx 8081
numprocs=1
redirect_stderr=true
stdout_logfile=/var/log/ngrok-webcam.log
user=pi
```

rereadして作成した設定ファイルをSupervisorに読み込ませます。

```
$ sudo supervisorctl reread
ngrok-webcam: available
```

addでSupervisorのサブプロセスにngrokを追加します。

```
$ sudo supervisorctl add ngrok-webcam
ngrok-webcam: added process group
$ sudo supervisorctl status ngrok-webcam
ngrok-webcam                       RUNNING    pid 30319, uptime 0:00:44
```

ngrokの[ダッシュボード](https://dashboard.ngrok.com/)のTunnels OnlineセクションにトンネルしているURLとクライアントのIPアドレスが表示されました。

![ngrok-webcam.png](/2015/08/08/raspberrypi-ngrok-motion/ngrok-webcam.png)

SafariからこのURLにアクセスするとMotionが配信している動画を見ることができます。


## freeboardに統合する

[freeboard](https://github.com/Freeboard/freeboard)のダッシュボードにMotionの動画を埋め込むと、Chromeでも表示できるようになります。

### Paneの追加

ADD PANEをクリックしてPaneを追加します。

![avocado-add-pane.png](/2015/08/08/raspberrypi-ngrok-motion/avocado-add-pane.png)

Paneの工具アイコンをクリックしてPaneの設定を行います。名前は「アボカドライブ」とつけました。

### Widgetの追加

次にPaneのプラスアイコンをクリックしてWidgetを追加します。

![avocado-add-widget.png](/2015/08/08/raspberrypi-ngrok-motion/avocado-add-widget.png)

WidgetのプラスアイコンをクリックしてWidgetの編集をします。動画のフレーム画像はサーバー側で変更されるので、ダッシュボードではリフレッシュしません。

* TYPE: Picture
* IMAGE URL: ngrokがトンネルしているURL
* REFRESH EVERY: 0にしてリフレッシュしない

![avocado-edit-widget.png](/2015/08/08/raspberrypi-ngrok-motion/avocado-edit-widget.png)

### 完成

Hubotのライトをオン/オフするコマンドが正しく動作しているかライブ映像で確認できるようになりました。

![avocado-live.png](/2015/08/08/raspberrypi-ngrok-motion/avocado-live.png)
