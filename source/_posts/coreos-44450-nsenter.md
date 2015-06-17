title: "CoreOS 444.5.0 and nsenter before docker exec command in Docker 1.3.0"
date: 2014-10-29 21:45:34
tags:
 - CoreOS
 - nsenter
 - Docker
 - IDCFCloud
description: Docker 1.3.0 has been released on October 16, 2014 and this version is supported by CoreOS 472.0.0 via alpha channel. And Docker 1.3.0 comes with docker exec command which can do without nsenter. Before releasing Docker 1.3.0 via stable channel, nsenter is still useful. Althogh nsenter is there but not docker-enter supported, I'm going to install docker-enter and helper function on CoreOS. A little bit older version of CoreOS 410.2.0 is provided on IDCF Cloud, it's still necessary to this environment as of October 29, 2014.
---

Docker 1.3.0 has been released on October 16, 2014 and this version is supported by CoreOS 472.0.0 via alpha channel. And Docker 1.3.0 comes with `docker exec` command which can do without nsenter. Before releasing Docker 1.3.0 via stable channel, nsenter is still useful. Althogh nsenter is there but not `docker-enter` supported, I'm going to install `docker-enter` and helper function on CoreOS.
A little bit older version of CoreOS 410.2.0 is [provided](http://www.idcf.jp/cloud/spec/os.html) on IDCF Cloud, it's still necessary to this environment as of October 29, 2014.

<!-- more -->


### wite_files directive

[coreos-util](https://github.com/mikedanese/coreos-util) provides some helpful utilities. I make [docker-enter](https://github.com/mikedanese/coreos-util/blob/master/script/docker-enter) a reference to cloud-config. A minor tweak is $0 usage. I replace $0 with BASH_ARGV[0] otherwise it shows a error message below.

> dirname: invalid option -- b

``` yml cloud-config.yml
write_files:
  - path: /etc/profile.d/nse-function.sh
    permissions: 0755
    content: |
      function nse() {
        sudo nsenter --pid --uts --mount --ipc --net --target $(docker inspect --format="&#123;&#123; .State.Pid }}" $1)
      }
  - path: /etc/profile.d/docker-enter-function.sh
    permissions: 0755
    content: |
      function docker-enter() {
        if [ -e $(dirname "${BASH_ARGV[0]}")/nsenter ]; then
            # with boot2docker, nsenter is not in the PATH but it is in the same folder
            NSENTER=$(dirname "${BASH_ARGV[0]}")/nsenter
        else
            NSENTER=nsenter
        fi

        if [ -z "$1" ]; then
            echo "Usage: docker-enter CONTAINER [COMMAND [ARG]...]"
            echo ""
            echo "Enters the Docker CONTAINER and executes the specified COMMAND."
            echo "If COMMAND is not specified, runs an interactive shell in CONTAINER."
        else
            PID=$(docker inspect --format "&#123;&#123;.State.Pid}}" "$1")
            if [ -z "$PID" ]; then
                exit 1
            fi
            shift

            OPTS="--target $PID --mount --uts --ipc --net --pid --"

            if [ -z "$1" ]; then
                # No command given.
                # Use su to clear all host environment variables except for TERM,
                # initialize the environment variables HOME, SHELL, USER, LOGNAME, PATH,
                # and start a login shell.
                sudo "$NSENTER" $OPTS su - root
            else
                # Use env to clear all host environment variables.
                sudo "$NSENTER" $OPTS env --ignore-environment -- "$@"
            fi
        fi
      }
```


