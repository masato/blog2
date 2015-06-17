title: "State pip.installed found in sls xxx is unavailable"
date: 2014-11-16 10:13:43
tags:
 - Salt
 - pip
 - Ubuntu
 - do-release-upgrade
description: I've been upgrading my Salt cluster which was created a few months ago and left as it was. The problem is happened after sudo do-release-upgrade from Ubuntu 12.04 to 14.04 with salt-master and salt-minions. When I run sudo salt '*' state.highstate, it failed because pip.installed state is unavailable, although pip is installed successfully. After googled I found some related posts.
---

I've been upgrading my Salt cluster which was created a few months ago and left as it was. The problem is happened after `sudo do-release-upgrade` from Ubuntu 12.04 to 14.04 with salt-master and salt-minions. When I run `sudo salt '*' state.highstate`, it failed because pip.installed state is unavailable, although pip is installed successfully. After googled I found some related posts.

* [22.26.22.2. Reloading Modules](http://docs.saltstack.com/en/latest/ref/states/#reloading-modules)
* [Required package not found yet package is installed.](https://groups.google.com/forum/#!msg/salt-users/x3yX0cBasXw/XyaEGHsQ-nYJ)
* [On first run "State docker.xxx found in sls is unavailable"](https://github.com/saltstack/salt/issues/15803)
* [pip.installed state has insufficient pip detection #7659](https://github.com/saltstack/salt/issues/7659)

<!-- more -->

### It's happened before

As I wrote in [this post](/2014/09/12/salt-tutorials-openshift-3-prepare/) it's happened before. At that time I manually removed python-pip packege then re-installed via get-pip.py as a workaround.

``` bash
$ sudo apt-get remove python-pip
$ wget https://raw.github.com/pypa/pip/master/contrib/get-pip.py
$ sudo python get-pip.py
```

### This SLS didn't work

This SLS file occurs a error. It looks correct at first sight.

``` yml /srv/salt/docker/init.sls
...
python-pip:
  pkg.installed

docker-py:
  pip.installed:
    - require:
      - pkg: pkg-core
      - pkg: python-pip
```

### A revised SLS

In found a related Salt Documatation in [22.26.22.2. Reloading Modules](http://docs.saltstack.com/en/latest/ref/states/#reloading-modules). It tells me how to solve this problem using `reload_modules` flag.

``` yml pep8.sls
python-pip:
  cmd:
    - run
    - cwd: /
    - name: easy_install --script-dir=/usr/bin -U pip
    - reload_modules: true

pep8:
  pip.installed
  requires:
    - cmd: python-pip
```

I revised my SLS as below.

``` yml /srv/salt/docker/init.sls
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
```

After purged old package, it works this time.

``` bash
$ sudo salt '*' pkg.purge python-pip
$ sudo salt '*' pkg.refresh_db
$ sudo salt '*' state.highstate
```