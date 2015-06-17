title: "Fleet and Systemd Unit section references"
date: 2014-11-15 14:13:12
tags:
 - fleet
 - systemd
description: It seems that my fleet units have been running after I switched to OpenResty as a dynamic routing unit. It might cause I revmoved Requires or changed to Wants derective. By the way, it is very difficult to understand of systemd units behaviour coupled with etcd and fleet. In a Unit section there are some directives related to dependencies and ordering. I confused differences between effectiveness, so I collect some references for study.
---

It seems that my fleet units have been running after I switched to OpenResty as a dynamic routing unit. It might cause I revmoved `Requires` or changed to `Wants` derective. By the way, it is very difficult to understand of systemd units behaviour coupled with etcd and fleet. In a Unit section there are some directives related to dependencies and ordering. I confused differences between effectiveness, so I collect some references for study.

<!-- more -->

### systemd man pages

* [systemd.unit](http://www.freedesktop.org/software/systemd/man/systemd.unit.html)
* [systemd.service](http://www.freedesktop.org/software/systemd/man/systemd.service.html)

#### DigitalOcean developer-friendly

* [How to Create Flexible Services for a CoreOS Cluster with Fleet Unit Files](https://www.digitalocean.com/community/tutorials/how-to-create-flexible-services-for-a-coreos-cluster-with-fleet-unit-files)

### In Japanese

* [Systemd入門(1) - Unitの概念を理解する](http://d.hatena.ne.jp/enakai00/20130914/1379146157)
* [Systemd入門(2) - Serviceの操作方法](http://d.hatena.ne.jp/enakai00/20130915/1379212787)
* [Systemd入門(3) - cgroupsと動的生成Unitに関する小ネタ](http://d.hatena.ne.jp/enakai00/20130916/1379295816)
* [Systemd入門(4) - serviceタイプUnitの設定ファイル](http://d.hatena.ne.jp/enakai00/20130917/1379374797)
* [Systemd入門(5) - PrivateTmpの実装を見る](http://d.hatena.ne.jp/enakai00/20130923/1379927579)
 
### BindsTo

* [How to start and stop a systemd unit with another?](http://serverfault.com/questions/622573/how-to-start-and-stop-a-systemd-unit-with-another)
* [Running Crate Data on CoreOS and Docker](https://zignar.net/2014/09/02/crate-data-on-core-os/)
* [How to deal with stale data when doing service discovery with etcd on CoreOS?](http://stackoverflow.com/questions/21597039/how-to-deal-with-stale-data-when-doing-service-discovery-with-etcd-on-coreos)
* [My backup strategy](http://morrisjobke.de/2014/01/09/My-backup-strategy/)

### After

* [Cause a script to execute after networking has started?](http://unix.stackexchange.com/questions/126009/cause-a-script-to-execute-after-networking-has-started)
* [Zero Downtime Frontend Deploys with Vulcand on CoreOS](https://coreos.com/blog/zero-downtime-frontend-deploys-vulcand/)
* [systemd how can I get a service to start after network is up](http://www.gossamer-threads.com/lists/gentoo/user/287444)