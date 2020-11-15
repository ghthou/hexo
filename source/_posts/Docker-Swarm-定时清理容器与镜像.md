---
title: Docker Swarm 定时清理容器与镜像
tags:
  - Docker
categories:
  - Docker
date: 2020-11-15 20:42:00
---

# 前言

在 Docker Swarm 环境中, 因服务更新, 迁移, 重启等操作, 我们会产生大量无用镜像与容器
 
如果不及时清理的话, 镜像会快速增长, 导致占满磁盘空间

理论上我们可以在每个节点配置一个清理的定时任务, 但是新增节点及更新定时任务配置的时候会不太方便

此时我们可以使用 Docker Swarm 的 global mode 为所有的 Docker 节点启动一个清理服务副本, 实现新节点加入时自动启动与配置更新时自动同步到所有节点

# 服务配置

## docker-compose.yml

创建 docker-compose.yml 文件, 内容如下

```yaml
# docker-compose 文件版本, 需与 docker engine 兼容, 否则启动失败
# https://docs.docker.com/compose/compose-file/#compose-and-docker-compatibility-matrix
version: "3.8"

# 配置时区为宿主机时区
x-bind-timezone:
  - &bind-localtime
    type: bind
    source: /etc/localtime
    target: /etc/localtime
    read_only: true
  - &bind-timezone
    type: bind
    source: /etc/timezone
    target: /etc/timezone
    read_only: true

services:
  docker-prune:
    # 请将镜像 tag 修改为宿主机的 docker 版本
    image: docker:19.03.13
    # https://docs.docker.com/engine/swarm/services/#create-services-using-templates
    hostname: "docker-prune-{{.Node.Hostname}}"
    entrypoint: docker-entrypoint.sh
    volumes:
      # 保持时区与宿主机一致(北京时间)
      - *bind-localtime
      - *bind-timezone
      - type: bind
        source: /var/run/docker.sock
        target: /var/run/docker.sock
    configs:
      - source: docker-entrypoint
        target: /usr/local/bin/docker-entrypoint.sh
        # 赋予执行权限
        mode: 0555
      - source: run
        target: /usr/local/bin/run.sh
        # 赋予执行权限
        mode: 0555
    environment:
      # cron 表达式, 必配 https://crontab.guru/
      CRON: "0 3 * * *"
      # 清理匹配到的镜像, 选配
      # 一般用于在自己或公司名下的项目镜像
      # 因为该类镜像会频繁的发布与更新, 而更新后上个版本理论上可以删除
      # 如果不配置或者为空则忽略该步骤
      PRUNE_IMAGE_BY_GREP: abc
    deploy:
      # global 模式, 在所有的 docker 节点上都会启动一份副本, 以达到 docker 节点新增时自动启动
      mode: global
# 在此处配置使用的网络, 如果不配置, docker 会创建一个默认网络 docker-prune_default
# 建议先创建一个外部网络, 然后使用改网络
#    networks:
#      swarm_webnet:
#
#networks:
#  swarm_webnet:
#    # 是否为外部网络
#    external: true

configs:
  docker-entrypoint:
    file: ./docker-entrypoint.sh
  run:
    file: ./run.sh
```

其中会用到两个脚本, 需与 docker-compose.yml 在同一文件夹中

## docker-entrypoint.sh

```sh
#!/bin/sh

# https://blog.knoldus.com/running-a-cron-job-in-docker-container/

# Start the run once job.
echo "Docker container has been started"

touch /var/log/cron.log

# Setup a cron schedule
echo "${CRON} run.sh >> /var/log/cron.log 2>&1
# This extra line makes it a valid cron" >scheduler.txt

# 配置定时任务
crontab scheduler.txt
# 后台启动定时任务
crond
# 查看定时任务执行日志
tail -f /var/log/cron.log
```

## run.sh

```sh
#!/bin/sh

# 清理容器
echo -e "\n清理容器 begin"
docker container prune --force
echo "清理容器 end"

# 清理镜像
echo -e "\n清理悬空镜像 begin"
docker image prune --force
echo "清理悬空镜像 end"

# 清理匹配到的镜像
if [ "${PRUNE_IMAGE_BY_GREP}" != "" ]; then
  # 如果镜像被使用, 则删除失败
  echo -e "\n清理匹配到的镜像 begin"
  docker images | grep "${PRUNE_IMAGE_BY_GREP}" | awk '{print $3 }' | xargs docker rmi
  echo "清理项目镜像 end"
fi
```

## 命令

### 启动

```console
docker stack deploy --compose-file docker-compose.yml docker-prune
```

### 停止

```console
docker stack rm docker-prune
```

### 更新/重启

```console
# 先停止, 再启动, 因为使用到了 docker config, 无法直接更新
docker stack rm docker-prune && docker stack deploy --compose-file docker-compose.yml docker-prune
```
