---
title: wercker本地部署
date: 2018-05-15 10:32:47
tags: docker
---
wercker本地部署

##安装docker

##安装Docker-Machine
```
$ base=https://github.com/docker/machine/releases/download/v0.14.0 &&
  curl -L $base/docker-machine-$(uname -s)-$(uname -m) >/tmp/docker-machine &&
  sudo install /tmp/docker-machine /usr/local/bin/docker-machine
```

##配置一下你的shell，以便查看
```
$ base=https://raw.githubusercontent.com/docker/machine/v0.14.0 &&
sudo wget "$base/contrib/completion/bash/docker-machine-prompt.bash" -P /etc/bash_completion.d &&
sudo wget "$base/contrib/completion/bash/docker-machine-wrapper.bash" -P /etc/bash_completion.d &&
sudo wget "$base/contrib/completion/bash/docker-machine.bash" -P /etc/bash_completion.d &&
source /etc/bash_completion.d/docker-machine-prompt.bash
```
修改你的~/.bashrc, 在环境变量PS1后追加`$(__docker_machine_ps1)`,如：
```
PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]$(__docker_machine_ps1)\$ '
```

##docker-machine的删除
```
docker-machine rm -f $(docker-machine ls -q) //you might need to use -force on Windows
rm $(which docker-machine)
```
##安装wercker-cli
```
sudo curl -L https://s3.amazonaws.com/downloads.wercker.com/cli/stable/linux_amd64/wercker -o /usr/local/bin/wercker
sudo chmod u+x /usr/local/bin/wercker
docker-machine create --driver virtualbox dev
eval "$(docker-machine env dev)"
```

```
http://devcenter.wercker.com/docs/quickstarts/building/golang
go get github.com/wercker/getting-started-golang
cd $GOPATH/src/github.com/wercker/getting-started-golang/
```

执行`wercker dev --expose-ports`的时候报错
```
$ wercker dev --expose-ports
--> Executing pipeline
--> Copied working directory: 0.00s
--> Running step: setup environment
Pulling from library/golang: latest
Digest: sha256:2ffa2f093d20c46e86435626f11bf163797400cf8f7cf14ecdc6403f1930045c
Status: Image is up to date for golang:latest
--> Copying source to container
--> Running step: wercker-init
/bin/bash: line 46: /pipeline/wercker-init-0b7cf4a4-44aa-41c6-bfd5-ffdd4608846a/run.sh: No such file or directory
--> Command exited with exit code: 1
--> Step failed: wercker-init Command exited with exit code: 1 0.02s
--> Pipeline failed: 4.90s
FATAL Step failed: wercker-init
```
