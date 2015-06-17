title: 'Ubuntu14.04のbyobuのドットファイル'
date: 2014-06-09 19:31:50
tags:
 - Ubuntu
 - byobu
 - tmux
 - Dokerfile
 - Putty
description: Ubuntu14.04のbyobuをputtyから使うで作成したDockerfileでは、byobuの初期設定をコンテナに接続してから手動で行っていました。Dockerfileで.emacs.dなどのドットファイルをADDするようにしたので、~/.byobu/ディレクトリもコピーするようにします。
---
[Ubuntu14.04のbyobuをputtyから使う](/2014/05/26/ubuntu-byobu-tmux-putty/)で作成したDockerfileでは、byobuの初期設定をコンテナに接続してから手動で行っていました。
Dockerfileで.emacs.dなどのドットファイルをADDするようにしたので、`~/.byobu/`ディレクトリもコピーするようにします。

<!-- more -->

### .byobuの設定ファイル

Dockerfileで初期設定としてADDする.byobuのドットファイルです。

``` bash
$ tree  ~/docker_apps/phusion/dotfiles/.byobu
dotfiles/.byobu/
├── backend
├── keybindings.tmux
└── status
```

byobuを使う場合、いつも以下のように設定しています。
* バックエンドはtmuxを使う 
* `ctrl+a`は、Emacsモードにして行頭にカーソルを移動する
* `ctrl+t`を、エスケープシーケンスにする
* ステータス通知にip_addressを表示する
* ステータス通知にlogoを表示しない、表示するとputtyでbyobuが毎秒スクロールしてしまう

backendにtmuxの指定をします。
``` bash ~/docker_apps/phusion/dotfiles/.byobu/backend
BYOBU_BACKEND=tmux
```

backendにtmuxをしたときのキーバインドの変更です。
``` bash ~/docker_apps/phusion/dotfiles/.byobu/keybindings.tmux
unbind-key -n C-a
unbind-key -n C-t
set -g prefix ^T
set -g prefix2 ^T
bind t send-prefix
```

画面下に表示されるステータスバーの設定を調整します。
``` bash ~/docker_apps/phusion/dotfiles/.byobu/status
# Status beginning with '#' are disabled.

# Screen has two status lines, with 4 quadrants for status
screen_upper_left="color"
screen_upper_right="color whoami hostname ip_address menu"
screen_lower_left="color logo distro release #arch session"
screen_lower_right="color network #disk_io custom #entropy raid reboot_required updates_available #apport #services #mail users uptime #ec2_cost #rcs_cost #fan_speed #cpu_temp battery wifi_quality #processes load_average cpu_count cpu_freq memory #swap #disk #time_utc date time"

# Tmux has one status line, with 2 halves for status
tmux_left=" #logo #distro release #arch session"
# You can have as many tmux right lines below here, and cycle through them using Shift-F5
tmux_right=" #network #disk_io #custom #entropy raid reboot_required updates_available #apport #services #mail #users uptime #ec2_cost #rcs_cost #fan_speed #cpu_temp #battery #wifi_quality #processes load_average cpu_count cpu_freq memory #swap #disk #whoami #hostname ip_address #time_utc date time"
#tmux_right="network #disk_io #custom entropy raid reboot_required updates_available #apport #services #mail users uptime #ec2_cost #rcs_cost fan_speed cpu_temp battery wifi_quality #processes load_average cpu_count cpu_freq memory #swap #disk whoami hostname ip_address #time_utc date time"
#tmux_right="network #disk_io custom #entropy raid reboot_required updates_available #apport #services #mail users uptime #ec2_cost #rcs_cost #fan_speed #cpu_temp battery wifi_quality #processes load_average cpu_count cpu_freq memory #swap #disk #whoami #hostname ip_address #time_utc date time"
#tmux_right="#network disk_io #custom entropy #raid #reboot_required #updates_available #apport #services #mail #users #uptime #ec2_cost #rcs_cost fan_speed cpu_temp #battery #wifi_quality #processes #load_average #cpu_count #cpu_freq #memory #swap whoami hostname ip_address #time_utc disk date time"
```

### Dockerfile

開発環境を構築するDockerfileからbyobuまわりの抜粋です。
``` bash ~/docker_apps/phusion/Dockerfile
## Development Environment
RUN apt-get install -y emacs24-nox emacs24-el git byobu wget curl unzip

## byobu
RUN mkdir -p /root/.byobu

## dotfiles
RUN mkdir -p /root/tmp
ADD dotfiles /root/
```

Dockerfileを修正したらタグのバージョンをあげてビルドします。latestのタグも付け替えます
``` bash
$ docker build -t masato/baseimage:1.2 .
$ docker tag masato/baseimage:1.2 masato/baseimage:latest
```

latestのイメージを使ってコンテナを起動します。
``` bash
$ docker run -i -t --rm --name dart -v ~/docker_apps/workspaces/dart_apps:/root/dart_apps \
  masato/baseimage /sbin/my_init /bin/bash
```

### コンテナから設定のバックポート

コンテナはdisposableなので、コンテナで作業していて気に入った設定ができたら、Dockerホストへコピーします。

``` bash
$ cd  ~/docker_apps/phusion
$ sudo docker cp dart:/root/.byobu/backend ./dotfiles/.byobu/
$ sudo docker cp dart:/root/.byobu/status ./dotfiles/.byobu/
$ sudo docker cp dart:/root/.byobu/keybindings.tmux ./dotfiles/.byobu/
$ sudo chown -R {ユーザー}:{グループ} ./dotfiles/.byobu/
```

### git commit

最後にDockerfileを`git commit`してリモートリポジトリにpushします。
``` bash
$ git add .
$ git commit -m 'my favorite dotfiles'
$ git push
```


