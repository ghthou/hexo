---
title: 我的 Linxu(CentOS) 初始安装后配置
author: ghthou
date: 2018-09-22 17:27:55
tags:
  - Linxu
categories:
  - Linxu
---

[TOC]

### 基础配置

主要使用阿里云的服务

#### DNS

```bash
echo "nameserver 223.5.5.5" | tee /etc/resolv.conf
echo "nameserver 223.6.6.6" | tee -a /etc/resolv.conf
```

#### yum

1. 备份

```bash
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
```

2. 下载新的CentOS-Base.repo 到/etc/yum.repos.d/

   - CentOS 5

     ```bash
     wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-5.repo
     # 或者
     curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-5.repo
     ```

   - CentOS 6

     ```bash
     wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repo
     # 或者
     curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repo
     ```

   - CentOS 7

     ```bash
     wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
     # 或者
     curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
     ```

3. 生成缓存

   ```bash
   yum clean all
   yum makecache
   ```

#### NTP

```bash
echo "driftfile  /var/lib/ntp/drift" | tee /etc/ntp.conf
echo "pidfile   /var/run/ntpd.pid" | tee -a /etc/ntp.conf
echo "logfile /var/log/ntp.log" | tee -a /etc/ntp.conf
echo "restrict    default kod nomodify notrap nopeer noquery" | tee -a /etc/ntp.conf
echo "restrict -6 default kod nomodify notrap nopeer noquery" | tee -a /etc/ntp.conf
echo "restrict 127.0.0.1" | tee -a /etc/ntp.conf
echo "server 127.127.1.0" | tee -a /etc/ntp.conf
echo "fudge  127.127.1.0 stratum 10" | tee -a /etc/ntp.conf
echo "server ntp.aliyun.com iburst minpoll 4 maxpoll 10" | tee -a /etc/ntp.conf
echo "restrict ntp.aliyun.com nomodify notrap nopeer noquery" | tee -a /etc/ntp.conf

sudo systemctl restart ntpd.service
```

#### 用户

```bash
useradd prod
```

#### 熵

```bash
sudo cat /proc/sys/kernel/random/entropy_avail
sudo yum install -y haveged
sudo systemctl start haveged
sudo systemctl enable haveged
sudo systemctl status haveged
sudo systemctl list-unit-files | grep haveged
sudo cat /proc/sys/kernel/random/entropy_avail
```

### 软件

#### [mcedit](https://midnight-commander.org/)

##### 说明

终端编辑器

##### 安装

```bash
sudo yum install -y mc
```

#### [ncdu](https://dev.yorhel.nl/ncdu)

##### 说明

可视化的空间分析程序

##### 安装

```bash
sudo wget -O /tmp/ncdu.tag.gz https://dev.yorhel.nl/download/ncdu-linux-i486-1.13.tar.gz 
sudo tar xf /tmp/ncdu.tag.gz -C /usr/local/bin/
sudo chmod +x /usr/local/bin/ncdu
```

#### [tldr](https://github.com/pepa65/tldr-bash-client)

##### 说明

命令行帮助手册

##### 安装

```bash
loc=/usr/local/bin/tldr  # elevated privileges needed for some locations
sudo wget -O $loc https://4e4.win/tldr
sudo chmod +x $loc
```

#### [shell-safe-rm](https://github.com/kaelzhang/shell-safe-rm)

##### 说明

安全的`rm`命令

##### 安装

```bash
sudo wget -O /tmp/safe-rm.sh https://github.com/kaelzhang/shell-safe-rm/blob/master/bin/rm.sh
sudo cp /tmp/safe-rm.sh /usr/local/bin/rm
sudo chmod 755 /usr/local/bin/rm
sudo cp /usr/bin/rm /usr/local/bin/rmo
```

#### [ctop](https://github.com/bcicen/ctop)

##### 说明

类似`top`的docker 容器指标界面

##### 安装

```bash
sudo wget -O /usr/local/bin/ctop https://github.com/bcicen/ctop/releases/download/v0.7.1/ctop-0.7.1-linux-amd64 
sudo chmod +x /usr/local/bin/ctop
```

#### [docker](https://www.docker.com/)

##### [docker-ce](https://docs.docker.com/install/linux/docker-ce/centos/)

```bash
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
# 使用阿里云的 yum 镜像
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo yum list docker-ce --showduplicates | sort -r  
sudo yum install docker-ce-18.06.1.ce
sudo systemctl start docker
sudo systemctl enable docker
docker version
# 自动补全 https://www.wilean.com/archives/359
sudo yum install -y bash-completion bash-completion-extras
# 镜像加速器
```

##### [docker-compose](https://docs.docker.com/compose/install/)

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
# 自动补全 https://docs.docker.com/compose/completion/
sudo curl -L https://raw.githubusercontent.com/docker/compose/1.22.0/contrib/completion/bash/docker-compose -o /etc/bash_completion.d/docker-compose
```

### 参考资料

[阿里云OPSX的 DNS,NTP 配置说明](https://opsx.alibaba.com/service?lang=zh-CN)

[阿里云内网 DNS 服务器地址](https://help.aliyun.com/knowledge_detail/40719.html#h2-u6392u67E5u65B9u6CD53)

[阿里公共 DNS 配置说明](http://www.alidns.com/setup/#linux)

[阿里云 docker 镜像](https://yq.aliyun.com/articles/110806)

