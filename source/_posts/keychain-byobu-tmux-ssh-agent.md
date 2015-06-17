title: "Keychain for ssh-agent with byobu and tmux"
date: 2014-10-16 01:16:05
tags:
 - byobu
 - tmux
 - ssh-agent
 - keychain
description: I used to evaluate ssh-agent output each time when I open new shells with byobu and tmux windows. It's because ssh-agent envorinment variables are not passed overa to new shell. I found that more than 10 ssh-agent processes are running. After Googling I reached a great post titled Understanding ssh-agent and ssh-add. 
---

I used to evaluate ssh-agent output each time when  I open new shells with byobu and tmux windows. It's because ssh-agent envorinment variables are not passed over to new shell. I found that more than 10 ssh-agent processes are running. After Googling I reached a great post titled [Understanding ssh-agent and ssh-add](http://blog.joncairns.com/2013/12/understanding-ssh-agent-and-ssh-add/). 

<!-- more -->


### keychain

For using same ssh-agent process with multiple shells or tmux windows, it should be reset `SSH_AUTH_SOCKET` enviroment inside that shell finding a ssh-agent sokcet which is already run.

I found [ssh-find-agent](https://github.com/wwalker/ssh-find-agent) project for the first time. Following the instruction it is recommended to use [keychain](https://github.com/funtoo/keychain) command because ssh-find-agent has reached its end of life.

It is possible to install with apt-get install.

``` bash
$ sudo apt-get install keychain
```

### Geneating a SSH key for GitHub

I genarated a new ssh key for GitHub and adding the publice key on my account page.

``` bash
$ ssh-keygen -t rsa -f ~/.ssh/github -C 'ma6ato@gmail.com for github'
```

To evaluate keychain command in .profike, it is enabled to add keys to a agent. When I login or sourcing .profile it is required to enter the passphrase each keys.

``` bash
$ echo 'eval `keychain --eval --agents ssh github`' >> ~/.profile
$ source ~/.profile
```

I verified my key by entering ssh command.
 
``` bash
$ ssh -T git@github.com
```

Finally I could do a git clone.

``` bash
$ cd ~/docker_apps
$ git clone git@github.com:masato/docker-lloyd.git
...
remote: Total 17 (delta 0), reused 0 (delta 0)
Checking connectivity... done.
```
