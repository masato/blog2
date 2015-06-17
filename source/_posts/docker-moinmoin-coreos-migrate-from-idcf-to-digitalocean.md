title: "MoinMoin in Production on CoreOS - Part7: IDCF Cloud to DigitalOcean"
date: 2014-10-23 02:55:51
tags:
 - MoinMoin
 - CoreOS
 - fleet
 - IDCFCloud
 - IDCFObjectStorage
 - DigitalOcean
 - ngrok
description: This is my first attempt at trying multi-cloud deployment with CoreOS. Not so famous cloud providers such as IDCF Cloud, it is very important to have cloud interoperabilities. With help of CoreOS and fleet, it would be possible to try out CoreOS on IDCF Cloud and deploy to any other clouds in production. What is more important is that without CloudFormation or OpenStack Heat, it would be possible to describe a template of muptiple instances also.
---

This is my first attempt at trying multi-cloud deployment with CoreOS. Not so famous cloud providers such as IDCF Cloud, it is very important to have cloud interoperabilities. With help of CoreOS and fleet, it would be possible to try out CoreOS on IDCF Cloud and deploy to any other clouds in production. What is more important is that without CloudFormation or OpenStack Heat, it would be possible to describe a template of muptiple instances also.

<!-- more -->

### docker-registry through ngrok

I am usually running a private docker-registry with IDCF Object Storage backend and I want to avoid opening public connection to my repositories. The good news is that through a ngrok tunnel I can allow remote connections from the internet if necessary.

I have A docker-registry container running with -e flags for IDCF Object Storage credentials.

``` bash
$ docker run -d -p 5000:5000 \
  -v /home/mshimizu/registry_conf:/registry_conf \
  -e DOCKER_REGISTRY_CONFIG=/registry_conf/config.yml \
  -e AWS_S3_ACCESS_KEY="{access_key}" \
  -e AWS_S3_SECRET_KEY="{secret_key}" |
  -e REGISTRY_SECRET=`openssl rand -base64 64 | tr -d '\n'` \
  registry:0.6.9
```

And then I run ngrok container pointing to that docker-registry.

``` bash
$ docker run -it --rm --name ngrok \
  wizardapps/ngrok:latest ngrok localhost:5000
```

Through ngrok provided URL it exposes private repositories to the internet.

### Dedicated etcd instance opened to a CoreOS cluster

In reference to [Setup a Dedicated etcd Cluster](https://github.com/kelseyhightower/kubernetes-fleet-tutorial#setup-a-dedicated-etcd-cluster) post, I decided to create a dedicated etcd instance.

I launched new IDCF Cloud instance from CoreOS ISO. After the instance started I opened console window in the browser, and changed `core` user password for enabling ssh login with password.

Then I could do ssh login to my new CoreOS instance with -o flag.

``` bash
$ ssh -A core@10.1.0.246 -o PreferredAuthentications=password
```

`210.129.xxx.xxx` is  a public ip address which is assigned to my account. And do not forget to add new allow port rules and port forwading rules to the firewall.

The point is that I create allow rules to port 22, 4001 and 7001 from a few soruces below.

* Oddly enough, 10.1.0.0/22 is necessary for connecting to his public ip address (in this case 210.129.xxx.xxx) from inside IDCF Cloud instances.
* Peer DigitalOcan droplet public ip address

I prepared cloud-config.yml file for dedicated etcd instance. It is disabled running etcd.service and docker.service using `mask: true`.

``` yml ~/cloud-config.yml
#cloud-config

hostname: etcd-do

coreos:
  fleet:
    etcd_servers: http://127.0.0.1:4001
    metadata: role=etcd
  etcd:
    name: etcd-do
    addr: 210.129.xxx.xxx:4001
    bind-addr: 0.0.0.0
    peer-addr: 210.129.xxx.xxx:7001
    cluster-active-size: 1
    snapshot: true
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
    - name: docker.service
      mask: true
    - name: timezone.service
      command: start
      content: |
        [Unit]
        Description=Set the timezone

        [Service]
        ExecStart=/usr/bin/timedatectl set-timezone Asia/Tokyo
        RemainAfterExit=yes
        Type=oneshot
ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1yc2EA...
```

I installed CoreOS to the local disk with cloud-config.yml file specified with the -c flag.

``` bash
$ sudo coreos-install -d /dev/sda -V 444.5.0 -C stable -c ./cloud-config.yml
```

### fleetctl testing

Before testing Core cluster successfully created, export two environment variables pointing to dedicated etcd instance. I could get a list of the cluster members.

``` bash
$ export FLEETCTL_ENDPOINT=http://210.129.xxx.xxx:4001
$ export FLEETCTL_TUNNEL=210.129.xxx.xxx
$ fleetctl list-machines
MACHINE         IP              METADATA
2b27d51f...     10.1.0.246      role=etcd
4a236dee...     10.1.1.191      role=moin
672419a9...     128.199.xxx.xxx role=moin
```

Additionally, I verified that all the member machines accessible via SSH.

``` bash 
$ fleetctl ssh 2b27d51f
$ fleetctl ssh 4a236dee
$ fleetctl ssh 672419a9
```

### Run a MoinMoin Service on a CoreOS

Following the instructions of [DigitalOcean tutorial](https://www.digitalocean.com/community/tutorials/how-to-create-and-run-a-service-on-a-coreos-cluster), I edited a template unit file.

``` bash ~/docker_apps/moin/moin@.service
[Unit]
Description=MoinMoin Service
After=docker.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill moin%i
ExecStartPre=-/usr/bin/docker rm moin%i

ExecStartPre=/bin/bash -c '/usr/bin/docker start || /usr/bin/docker run -v /usr/local/share/moin/data/pages --name moin-vol xxxxxxxx.ngrok.com/moin-data true && true'
ExecStartPre=/usr/bin/docker pull xxxxxxxx.ngrok.com/moin

ExecStart=/usr/bin/docker run --name moin%i --volumes-from moin-vol -p 80:80 xxxxxxxx.ngrok.com/moin
ExecStop=/usr/bin/docker stop moin%i

[Install]
WantedBy=multi-user.target

[X-Fleet]
X-Conflicts=moin@*.service
MachineMetadata=role=moin
```

And I submitted fleet unite template to my cluster.

``` bash
$ fleetctl submit moin@.service
$ fleetctl list-unit-files
UNIT            HASH    DSTATE          STATE           TARGET
moin@.service   6e5fc67 inactive        inactive        -
```

Then I could load and start MoinMoin service with the port number.

``` bash
$ fleetctl load moin@80.service
Unit moin@80.service loaded on 4a236dee.../10.1.1.191
$ fleetctl start moin@80.service
Unit moin@80.service launched on 4a236dee.../10.1.1.191
$ fleetctl list-unit-files
UNIT            HASH    DSTATE          STATE           TARGET
moin@.service   6e5fc67 inactive        inactive        -
moin@80.service 6e5fc67 launched        launched        4a236dee.../10.1.1.191
$ fleetctl list-units
UNIT            MACHINE                 ACTIVE  SUB
moin@80.service 4a236dee.../10.1.1.191  active  running
$ fleetctl journal -f moin@80.service
```

### failover testing

Next thing should I do is to stop running CoreOS node without hesitation.

``` bash
$ fleetctl ssh moin@80.service
$ sudo shutdown -h now
```

The 10.1.1.191 node previously running moin@80.service was disappeared in the list. In place of stopped node on IDCF Cloud, a new container on DigitalOcean activated being a failover node. 

``` bash
$ fleetctl list-machines
MACHINE         IP              METADATA
2b27d51f...     10.1.0.246      role=etcd
672419a9...     128.199.xxx.xxx role=moin
$ fleetctl list-units
UNIT            MACHINE                         ACTIVE  SUB
moin@80.service 672419a9.../128.199.xxx.xxx     active  running
$ fleetctl journal -f moin@80.service
```
