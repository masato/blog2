title: packer-docker-ansible
tags:
---



Dockerfile を使用せずに、Ansible を使用して Docker イメージを作成
Packer
docker containerに対して直接ansibleを実行する
http://tdoc.info/blog/2014/04/08/ansible_docker_connection.html
dockerのイメージを構築するにはDockerfileを使って構築します。しかし、Dockerfileはほぼ単なるshell scriptなのでいろいろと書きにくいという問題があります。そのため、packerを使ってイメージを構築する手段が取られたりします。が、packerのansible provisionerはansible-pullを内部で実行するという形式のため、ansible実行環境やgitをイメージの中に入れる必要があります。

また、起動しているコンテナに対してコマンドを実行するためにはsshで入る必要があります。これはつまり、sshdを入れてsshのポートをexposeする必要があり、なおかつ動的に変わるdockerの外側sshポートを把握する必要があります。

これらの問題点を解決するために、ansibleが直接dockerコンテナとやりとりをするプラグインを作成しました。(注: ansibleはssh以外も使え、さらにその部分はプラグイン構造になっていて任意に追加できるのです)
Packer の provisioners に Ansible を指定して Docker イメージを作成する
http://qiita.com/Hexa/items/0896e4285ab993f8a499
Building VM images with Ansible and Packer
https://servercheck.in/blog/server-vm-images-ansible-and-packer
docker-ansible-packer
https://github.com/eggsby/docker-ansible-packer
ImmutableInfrastructure with Ansible and Packer
http://blog.codeship.io/2014/05/08/ansible-packer-immutable-infrastructure.html
PackerとAnsibleを使ってDockerイメージを作成する
http://blog.ieknir.com/blog/build-docker-image-using-packer-ansible-local-provisioner/
Manage Docker Containers using CoreOS - Part 3
http://2mohitarora.blogspot.jp/2014/05/manage-docker-containers-using-coreos_22.html