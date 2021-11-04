---
title: local_docker
toc: true
date: 2021-11-04 10:12:38
tags:
  - other
  - blog
categories:
  - other

---


```dockerfile
FROM ubuntu:21.04

LABEL maintainer="Thh"


RUN sed -i "s@http://.*archive.ubuntu.com@http://repo.huaweicloud.com@g" /etc/apt/sources.list &&  sed -i "s@http://.*security.ubuntu.com@http://repo.huaweicloud.com@g" /etc/apt/sources.list 
RUN apt-get update \
    && apt-get install -y curl zip unzip git supervisor sqlite3 zsh vim net-tools 

RUN git clone https://gitee.com/mirrors/oh-my-zsh.git ~/.oh-my-zsh \
    && cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc \
    && sed  -i "s@robbyrussell@af-magic@g" ~/.zshrc \
    &&  git config --global --add oh-my-zsh.hide-dirty 1

CMD ["/bin/sh","-c","while true; do   date   +'%Y-%m-%d %T %z'; sleep 2; done"]
```

```yml
version: '3.5'

networks:
  local_net:
    driver: bridge
services: #表示这是一组服务 
  ubuntu:
    build:
      context: ubuntu
    working_dir: /var/www
    volumes:
      - ${WORKSPACE}:/var/www:cached
    restart: always #docker服务重启后nginx的docker容器也重启
    networks:
      - local_net
```
<!--more-->


