---
title: 使用 Docker 快速搭建开发环境
tags:
  - Docker
categories:
  - Docker
date: 2018-01-13 11:14:00
---
> 在代码开发中, 除了语言开发环境及 IDE 外, 我们往往还需要依赖其他第三方服务, 如:` 数据库 `,` 服务器 `,` 缓存 `,` 搜索 `,`MQ` 等等. 而这些服务的安装各式各样, 有的极为复杂, 有的对开发机有极大的限制, 甚至有的直接不支持当前开发机. 给我们的开发环境搭建带来了极大的困难. 这时我们可以选择使用 `Docker` 来快速搭建开发环境, 屏蔽复杂的安装过程, 服务配置.

### 什么是 Docker
我们参考 Docker 官网中的概述 [what-docker](https://www.docker.com/what-docker)
> Docker 是世界领先的软件容器平台。** 开发人员使用 Docker 来消除与同事的代码协作时的 “我机器上的工作” 问题 **。运营商使用 Docker 在独立的容器中并行运行和管理应用程序，以获得更好的计算密度。企业使用 Docker 构建灵活的软件传送管道，可以更快，更安全地运行新功能，并且对于 Linux 和 Windows Server 应用程序都有信心。
-- 来自谷歌翻译

在其中的 `Docker For Developers` 部分中, 我们可以查看对于我们开发者具体有哪些作用
> Docker 自动执行设置和配置开发环境的重复任务，以便开发人员可以专注于重要的事情：构建出优秀的软件。

> 使用 Docker 的开发人员不必安装和配置复杂数据库，也不用担心在不兼容的语言工具链版本之间切换。当应用程序 Docker 化时，这种复杂性被推入容易构建，共享和运行的容器中。将同事加入新的代码库不再意味着安装软件和解释安装程序的时间。Dockerfiles 随附的代码更简单：依赖关系被拉为整齐的 Docker 映像，任何具有 Docker 和编辑器的人都可以在几分钟内构建和调试应用程序。
-- 来自谷歌翻译

** 简单来说, 使用 Docker 我们可以专注于代码的编写, 忽略其他软件复杂的安装, 配置. 同时可以统一线上, 线下环境, 不受服务版本差异的影响 **

### 安装 Docker
请参考 Docker 官方文档中的 [Install Docker](https://docs.docker.com/engine/installation/)
目前 Docker 支持的系统版本如下
![Docker 支持的系统版本](/images/使用-Docker-快速搭建开发环境/Docker支持的系统版本.png)
Docker 最初是在 Ubuntu 12.04 上开发实现的, 另外 Docker 官网文档中的一些操作命令也是基于 Ubuntu 来讲解的, 如果有条件, 推荐使用 Ubuntu 
Linux 安装完成后, 请查看 [Post-installation steps for Linux](https://docs.docker.com/engine/installation/linux/linux-postinstall/#manage-docker-as-a-non-root-user) 完成一些后续配置
对于 Linux 用户需要特别注意, 如果是以非 `root` 用户运行, 需要创建 `docker` 组, 并将当前用户添加到 `docker` 组中
```bash
$ sudo groupadd docker
$ sudo usermod -aG docker $USER
```
Docker 中使用的镜像都需要从网站上下载, 因为网络原因, 国内下载速度往往不佳, 此时可以使用国内的一些加速器来加速下载, 如:[DaoCloud](https://www.daocloud.io/mirror#accelerator-doc),[阿里云](https://cr.console.aliyun.com/#/accelerator), 具体用法, 请登录后查看网站说明文档

### 使用 Docker 搭建开发环境
现在以搭建 `mysql` 为例
#### 搜索 `mysql` 镜像
首先从 [hub.docker.com](https://hub.docker.com) 网站中搜索你需要的镜像, 如 `mysql`
![hub 搜索镜像](/images/使用-Docker-快速搭建开发环境/hub搜索镜像.png)
其中第一个带有 `official` 单词的表明为 Dcoker 官方提供的镜像, 下面的三个为个人 / 组织上传的镜像
我们点击右侧 `DETAILS` 按钮查看镜像详情
![hub 镜像说明](/images/使用-Docker-快速搭建开发环境/hub镜像说明.png)
图中的 `8.0.1` 至 `5.5.55` 四行表示支持的 `mysql` 版本, 同时附带镜像构建的 `Dockerfile` 文件
右侧的 `docker pull mysql` 是镜像的下载命令, 此时我们可以在命令行中执行该命令进行下载, 默认下载版本为 `latest`
如果希望指定下载版本, 使用如下命令格式 `docker pull mysql:版本号 `, 如 `docker pull mysql:5.6`

#### 下载 `mysql` 镜像
```bash
$ docker pull mysql:5.7
```
#### 运行 `mysql` 镜像
```bash
$ docker run --name mysql --rm -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root mysql:5.7
```
此时会在命令行中输出该容器运行时的日志, 若要退出, 请按 `Ctrl+c`
如果希望在后台运行, 加入 `-d` 参数即可
运行参数说明
```bash
--name mysql #镜像运行的容器名称为 mysql
--rm #容器退出后删除该容器
-p 3306:3306 #将本机的 3306 端口映射到该容器的 3306 端口
-e MYSQL_ROOT_PASSWORD=root #为容器配置一个名为 MYSQL_ROOT_PASSWORD, 值为 root 的环境变量, 因 mysql 容器的特殊性, 必须配置该环境变量
-d #在后台运行该容器
```
### 测试容器
在后台运行 `mysql` 容器
```bash
$ docker run --name mysql --rm -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7
```
查看当前运行容器列表
```bash
$ docker ps
```
![docker ps](/images/使用-Docker-快速搭建开发环境/docker-ps.png)
我们可以发现 `mysql` 已在后台运行
此时我们可以使用 `Navicat`,`SQLyog` 进行链接测试
`ip`: 运行容器机器的 ip
` 端口 `:3306
` 用户名 `:root
` 密码 `:root, 即 `MYSQL_ROOT_PASSWORD` 对应的值
亦可使用如下命令进入 `mysql` 命令行
```bash
$ docker run -it --link mysql:mysql --rm mysql:5.7 sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR"-P"$MYSQL_PORT_3306_TCP_PORT"-uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'
```
运行参数说明
```bash
-it #运行容器后进入一个交互式的终端
--link mysql:mysql #链接一个名称为 mysql 的容器, 并为该容器配置一个名为 mysql 的 hosts
sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR"-P"$MYSQL_PORT_3306_TCP_PORT"-uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"' #为运行容器后执行的命令, 其中诸如 $MYSQL_PORT_3306_TCP_ADDR,$MYSQL_PORT_3306_TCP_PORT 环境变量是容器根据 --link mysql:mysql 自动生成
```
#### 数据保存
mysql 镜像默认使用的配置文件为 `/etc/mysql/my.cnf`
如果我们需要自定义配置文件可以使用如下命令覆盖原本配置
```
$ docker run --name mysql --rm -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root -d -v ~/docker/data/mysql/my.cnf:/etc/mysql/my.cnf mysql:5.7
```
运行参数说明
```bash
-v ~/docker/data/mysql/my.cnf:/etc/mysql/my.cnf #使用当前机器下的 ~/docker/data/mysql/my.cnf 文件挂载为容器中的 /etc/mysql/my.cnf 文件
```
在 `mysql` 镜像中默认存储目录为 `/var/lib/mysql`, 这样存在容器删除后数据丢失的问题
为了防止这一情况产生, 我们需要将外部文件夹挂载到容器的 `/var/lib/mysql` 中
```bash
$ docker run --name mysql --rm -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root -d -v ~/docker/data/mysql/datadir:/var/lib/mysql mysql:5.7
```
此时我们查看 `~/docker/data/mysql/datadir` 文件夹
```bash
ll -h ~/docker/data/mysql/datadir
```
![datadir 文件夹](/images/使用-Docker-快速搭建开发环境/datadir文件夹.png)
发现已经在该文件夹内生成了一些 `mysql` 的初始化文件
关于 `mysql` 镜像的更多信息可在 [hub.docker.com](https://hub.docker.com) 中对应的 [镜像详情](https://hub.docker.com/_/mysql/) 查看
关于其他如 `redis`,`nginx`,`mongo` 等镜像的搭建及配置皆可在 [hub.docker.com](https://hub.docker.com) 中搜索查看

** 如果希望更加系统的学习 `Docker` 信息, 请查看 [官网文档](https://docs.docker.com/)**
如果想查看中文文档, 可以去看 [Docker —— 从入门到实践](https://www.gitbook.com/book/yeasy/docker_practice/details)

### 相关资料
- [Docker 官方文档](https://docs.docker.com/)
- [Docker —— 从入门到实践](https://www.gitbook.com/book/yeasy/docker_practice/details)
- [Docker-labs](https://github.com/docker/labs)