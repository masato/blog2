title: 'Salt docker.pulled state causes an APIError'
date: 2014-11-16 12:44:13
tags:
 - Salt
 - Ubuntu
 - do-release-upgrade
 - docker-py
description: After python-pip works correctly in a Salt cluster, I face yet another problem. This time salt.states.dockerio doesn't work. An error occured when I use docker.pulled state, a message tells that client and server don't have same version (client  1.15, server 1.14). It seems that this is also Ubuntu upgrade related problems.
---

After `python-pip` works correctly in a Salt cluster, I face yet another problem. This time `salt.states.dockerio` doesn't work. An error occured when I use `docker.pulled` state, a message tells that client and server don't have same version (client : 1.15, server: 1.14). It seems that this is also Ubuntu upgrade related problems.

<!-- more -->

### client and server don't have same version

A `docker.pulled` state causes an error because docker api versions are not same.

``` bash
$ sudo salt 'minion1*' state.highstate
...
----------
          ID: jpetazzo/nsenter
    Function: docker.pulled
      Result: False
     Comment: An exception occurred in this state: Traceback (most recent call last):
                File "/usr/lib/pymodules/python2.7/salt/state.py", line 1379, in call
                  **cdata['kwargs'])
                File "/usr/lib/pymodules/python2.7/salt/states/dockerio.py", line 253, in pulled
                  returned = pull(name, tag=tag)
                File "/usr/lib/pymodules/python2.7/salt/modules/dockerio.py", line 1646, in pull
                  client = _get_client()
                File "/usr/lib/pymodules/python2.7/salt/modules/dockerio.py", line 273, in _get_client
                  client._version = client.version()['ApiVersion']
                File "/usr/local/lib/python2.7/dist-packages/docker/client.py", line 1011, in version
                  return self._result(self._get(self._url("/version")), True)
                File "/usr/local/lib/python2.7/dist-packages/docker/client.py", line 93, in _result
                  self._raise_for_status(response)
                File "/usr/local/lib/python2.7/dist-packages/docker/client.py", line 89, in _raise_for_status
                  raise errors.APIError(e, response, explanation=explanation)
              APIError: 404 Client Error: Not Found ("client and server don't have same version (client : 1.15, server: 1.14)")
     Changes:
----------
          ID: nsenter-container
    Function: docker.installed
        Name: nsenter
      Result: False
     Comment: image "jpetazzo/nsenter" does not exist
     Changes:
```

I use a [pkg.latest](http://docs.saltstack.com/en/latest/ref/states/all/salt.states.pkg.html#salt.states.pkg.latest) state ensuring lxc-docker package is always latest. It seems that there is no problem.

``` yml /srv/salt/docker/init.sls
include:
  - common

python-pip:
  cmd:
    - run
    - cwd : /
    - name: easy_install --script-dir=/usr/bin -U pip
    - reload_modules: true

docker-py:
  pip.installed:
    - require:
      - pkg: pkg-core
      - cmd: python-pip

docker-dependencies:
  pkg.installed:
    - pkgs:
      - iptables
      - ca-certificates
      - lxc

docker-repo:
  pkgrepo.managed:
    - humanname: Docker Repo
    - name: deb https://get.docker.io/ubuntu docker main
    - key_url: https://get.docker.io/gpg
    - require:
      - pkgrepo: docker-repo
      - pkg: pkg-core

lxc-docker:
  pkg.latest:
    - require:
      - pkg: docker-dependencies

docker:
  service.running

jpetazzo/nsenter:
  docker.pulled:
    - name: jpetazzo/nsenter:latest

nsenter-container:
  docker.installed:
    - name: nsenter
    - image: jpetazzo/nsenter
    - volumes:
      - /usr/local/bin: /target

/usr/local/bin/nse:
  file.managed:
    - source: salt://docker/nse
    - mode: 775
```

### Docker is not updated

I realized that lxc-docker package was not updated after hit `docker version` command to a problematic minion. 

``` bash
$ sudo salt 'minion1.*' cmd.run 'docker version'
minion2.cs29dcloud.internal:
    Client version: 1.2.0
    Client API version: 1.14
    Go version (client): go1.3.1
    Git commit (client): fa7b24f
    OS/Arch (client): linux/amd64
    Server version: 1.2.0
    Server API version: 1.14
    Go version (server): go1.3.1
    Git commit (server): fa7b24f
```

Then I logged in to this minion and upgrade packages manually. Unfortunately, it said that everything was updated.

``` bash
$ sudo apt-get update
$ sudo apt-get upgrade
```

### apt sources.list is disabled after upgraded to 14.04

I checked an apt sources.list and found that docker repository was disabled when I executed `do-release-upgrade` to 14.04.
```
$ cat /etc/apt/sources.list.d/docker.list
# deb http://get.docker.io/ubuntu docker main # disabled on upgrade to trusty
```

Finally, a lxc-docker is updated and server API version has same 1.15 version with docker-py.

```
$ sudo salt-call pkg.refresh_db
$ docker version
Client version: 1.3.1
Client API version: 1.15
Go version (client): go1.3.3
Git commit (client): 4e9bbfa
OS/Arch (client): linux/amd64
Server version: 1.3.1
Server API version: 1.15
Go version (server): go1.3.3
Git commit (server): 4e9bbfa
```