---
title: 安装Vaultwarden代替lastpass
pubDatetime: 2021-07-05 16:09:46 +08:00
lastmod: 2026-01-26 11:04:03 +08:00
timezone: UTC+8
url: /posts/lastpass-to-vaultwarden/
description: 安装Vaultwarden代替lastpass
toc: true
tags:
  - bitwarden
  - lastpass
  - password
  - docker
---
多年来一直使用Lastpass作为主要的密码管理工具，直到最近LastPass 宣布将从今年 3 月 16 日开始限制免费版用户访问该服务的设备类型，限制如下：

> LastPass 允许用户在桌面电脑（笔记本或台式机上的浏览器）和移动设备（手机、平板电脑、智能手表等）上使用，新的措施将要求免费用户只能选择一种设备类型，但不会限制设备数量。

> 3 月 16 日开始，LastPass 免费版用户首次登陆时将被要求选择访问的设备类型，用户可以修改活跃设备类型 3 次，付费用户则没有这样的限制。

鉴于平时个人移动端和桌面端使用的是同一密码库，决定将密码库从`Lastpass`迁移到`Vaultwarden`，以方便自己控制。当然还有其他的密码管理器可以使用，比如`keepass`+`webdav`。

## 一、准备工作

鉴于Bitwarden官方镜像体积过大，且要求的服务器配置较高，决定使用第三方基于Rust开发的[Vaultwarden](https://github.com/dani-garcia/vaultwarden)来部署，该镜像与Bitwarden客户端兼容、硬件配置要求低、可以使用官方版本某些收费功能（TOTP），非常使用自托管部署。当然也缺失某些功能，具体缺失项请查看[官方Wiki](https://github.com/dani-garcia/vaultwarden/wiki#missing-features)，对于个人使用而言功能足够。

首先需要准备一台VPS主机或者家里的NAS。VPS主机配置不用太高，512MB内存即可，我使用的是Oracle Cloud韩国主机，为了使用域名访问，建议使用国外或者香港主机。

然后准备一个域名，最便宜的即可。

## 二、安装Docker

以下命令是基于Ubuntu系统的演示，其它的请参考官网：

>安装 Docker CE (社区版)：<https://docs.docker.com/engine/install/>

>安装 Docker Compose：<https://docs.docker.com/compose/install/#install-compose>

Ubuntu系统安装Docker CE 如下：

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
#测试一下有没有成功
sudo apt-key fingerprint 0EBFCD88
#有以下反馈就表示成功
pub   rsa4096 2017-02-22 [SCEA]
      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
sub   rsa4096 2017-02-22 [S]
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
#验证一下是不是正确安装
sudo docker run hello-world
#有以下反馈就表示正确安装
root@localhost:~# sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
0e03bdcc26d7: Pull complete 
Digest: sha256:4cf9c47f86df71d48364001ede3a4fcd85ae80ce02ebad74156906caff5378bc
Status: Downloaded newer image for hello-world:latest
Hello from Docker!
This message shows that your installation appears to be working correctly.
To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.
To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash
Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/
For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

Ubuntu系统安装 Docker Compose如下：

```bash
#安装 Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.30.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 检测一下是不是成功安装
docker-compose --version
```

## 三、安装Vaultwarden

建立需要的文件夹

```bash
mkdir -p /root/bitwarden/bw-data
mkdir -p /root/bitwarden/caddy-data/config
mkdir -p /root/bitwarden/caddy-data/data
mkdir -p /root/bitwarden/caddy-data/sites
mkdir -p /root/bitwarden/fail2ban/action.d
mkdir -p /root/bitwarden/fail2ban/filter.d
mkdir -p /root/bitwarden/fail2ban/jail.d
mkdir -p /root/bitwarden/logs

```

### 3.1 创建docker-compose配置文件

```bash
vim /root/bitwarden/docker-compose.yml
```

```yaml
services:
  vaultwarden:
    container_name: vaultwarden
    # image: bitwardenrs/server:latest
    image: vaultwarden/server:latest
    restart: always
    volumes:
      - ${PWD}/bw-data:/data
      - ${PWD}/logs:/logs
    environment:
      - IP_HEADER=X-Real-IP
      - DOMAIN=https://vw.domain.tld
      #- IP_HEADER=X-Forwarded-For
      - TZ=Asia/Shanghai
      - WEBSOCKET_ENABLED=true
      - SIGNUPS_ALLOWED=true
      #- SHOW_PASSWORD_HINT="false"
      #- ADMIN_TOKEN="some_random_token_as_per_above_explanation"
      - ROCKET_WORKERS=20
      #- DISABLE_ICON_DOWNLOAD='true'
      - LOG_FILE=/logs/bitwarden.log

  caddy:
    restart: always
    #Official Caddy 2.0 image
    image: "caddy:latest"
    container_name: caddy
    depends_on:
      - vaultwarden
    environment:
      - TZ=Asia/Shanghai
      #- LOG_FILE=/logs/caddy.log
      # Update this if SSL required according to the use of your own cert or requuest one from Let's Encrypt
      #- SSLCERTIFICATE=/path/to/ssl/fullcert.pem
      #- SSLKEY=/path/to/ssl/key.pem
      - ACMEE_AGREE='true'            # agree to Let's Encrypt Subscriber Agreement
      - DOMAIN=bitwarden.example.org              # CHANGE THIS! Used for Auto Let's Encrypt SSL
      - EMAIL=bitwarden@example.org              # CHANGE THIS! Optional, provided to Let's Encrypt
    ports:
      - 80:80        # Needed for the ACME HTTP-01 challenge.
      - 443:443
      - 443:443/udp  # Needed for HTTP/3.
    volumes:
      - ${PWD}/caddy-data/config/Caddyfile:/etc/caddy/Caddyfile
      - ${PWD}/caddy-data/data:/data
      - ${PWD}/caddy-data/sites:/var/www/html
      - ${PWD}/logs:/logs
      - caddycerts:/root/.caddy

  fail2ban:
       # Implements fail2ban functionality, banning ips that
       # try to bruteforce your vault
       # https://github.com/dani-garcia/bitwarden_rs/wiki/Fail2Ban-Setup
       # https://github.com/crazy-max/docker-fail2ban
    image: crazymax/fail2ban:latest
    restart: always
    container_name: fail2ban
    depends_on:
      - vaultwarden
    volumes:
      - ${PWD}/fail2ban:/data
      - ${PWD}/bw-data:/bitwarden:ro
      - ${PWD}/logs:/logs
    network_mode: "host"
    privileged: true
    cap_add:
      - NET_ADMIN
      - NET_RAW
    environment:
      - F2B_DB_PURGE_AGE=30d
      - F2B_LOG_TARGET=/logs/fail2ban.log
      - F2B_LOG_LEVEL=INFO
      - F2B_IPTABLES_CHAIN=INPUT
      - TZ=Asia/Shanghai

  watchtower:
    container_name: watchtower
    image: containrrr/watchtower:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - TZ=Asia/Shanghai
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_SCHEDULE=0 0 2 * * *
      #- WATCHTOWER_POLL_INTERVAL=86400
    restart: always

volumes:
  caddycerts:
```

启用了4个组件：`vaultwarden`、`caddy`、`fail2ban`、`watchtower`

其中caddy用于反代vaultwarden，fail2ban用于防止暴力破解，watchtower用于容器自动更新。配置文件中DOMAIN 和 EMAIL需要更改为需要绑定的域名和邮箱，以用于申请SSL证书。**如果服务器上有其他服务需要，可以将caddy独立配置，以方便后续代理其他服务。**

### 3.2 创建Caddy 配置文件Caddyfile

```bash
vim /root/bitwarden/caddy-data/config/Caddyfile
```

```bash
{$DOMAIN}:443 {
  # 容器中的 443 端口上的 Caddy 到 bitwarden_rs 私有实例
  # 如果 Caddy 暴露到网络中，则使用它
  #log {
  #       output file {$LOG_FILE} {
  #        roll_size 50MiB # https://caddyserver.com/docs/caddyfile/directives/log#log
  #        roll_keep 5 # https://caddyserver.com/docs/caddyfile/directives/log#log
  #    }
  #    level INFO
  #  }

  # 仅取消注释两行中的一行。取决于你提供自己的证书还是从 Let's Encrypt 请求证书
  # tls {$SSLCERTIFICATE} {$SSLKEY}
  tls {$EMAIL}

  encode zstd gzip

  header / {
       # 启用 HTTP Strict Transport Security (HSTS)
       Strict-Transport-Security "max-age=31536000;"
       # 启用 cross-site filter (XSS) 并告诉浏览器阻止检测到的攻击
       X-XSS-Protection "1; mode=block"
       # 禁止在框架内渲染站点 (clickjacking protection)
       X-Frame-Options "DENY"
       # 防止搜索引擎收录 (可选)
       X-Robots-Tag "none"
       # 移除服务器名称
       -Server
   }
  # 协商端点也被代理到 Rocket
  reverse_proxy /notifications/hub/negotiate vaultwarden:80

  # Notifications 重定向到 websockets 服务器
  reverse_proxy /notifications/hub vaultwarden:3012

  # 代理到 Rocket
  reverse_proxy vaultwarden:80 {
    # Send the true remote IP to Rocket, so that vaultwarden can put this in the
    # log, so that fail2ban can ban the correct IP.
    header_up Host {host}
    header_up X-Real-IP {remote_host}
    header_up X-Forwarded-For {remote_host}
    #header_up X-Forwarded-Proto {scheme}
  }
}
```

注意使用fail2ban的时候一定要真是IP地址给vaultwarden，否则日志中显示的IP地址不对，fail2ban无法正常工作。

根据caddy官方文档，`X-Forwarded-For` 和 `X-Forwarded-Proto` 标头不会默认传递，需要手动配置，且2.0后命令更改为 `header_up` 。

### 3.3 创建fail2ban配置文件

```bash
vim /root/bitwarden/fail2ban/action.d/iptables-common.local
```

```bash
# /root/bitwarden/fail2ban/action.d/iptables-common.local
[Init]
blocktype = DROP
[Init?family=inet6]
blocktype = DROP
```

```sh
vim /root/bitwarden/fail2ban/filter.d/vaultwarden.local
```

```bash
# path_f2b/filter.d/vaultwarden.local
[INCLUDES]
before = common.conf

[Definition]
failregex = ^.*Username or password is incorrect\. Try again\. IP: <ADDR>\. Username:.*$
ignoreregex =
```

```bash
vim /root/bitwarden/fail2ban/filter.d/vaultwarden-admin.local
```

```bash
# path_f2b/filter.d/vaultwarden-admin.local
[INCLUDES]
before = common.conf

[Definition]
failregex = ^.*Invalid admin token\. IP: <ADDR>.*$
ignoreregex =
```

如果在`CentOS 7`, `Fail2Ban v0.9.7` 环境中 fail2ban.log 日志中报错如下

```bash
fail2ban.filter [5291]: ERROR No 'host' group in '^.*Username or password is incorrect\. Try again\. IP: <ADDR>\. Username:.*$'
```

请使用 `<HOST>` 代替配置文件中的 `<ADDR>`

```bash
vim /root/bitwarden/fail2ban/jail.d/jail.local
```

```bash
# path_f2b/filter.d/jail.local
[DEFAULT]
bantime = 6h
findtime = 6h
maxretry = 5
# To enable email alerts, uncomment `destemail` and `sender` below,
# enter your destination and sender emails in the fields,
# and uncomment the `action_mwl` action in the jails below.
#destemail = email you want to send to
#sender = email you want to send from

[vaultwarden]
enabled = true
port = 80,443,8081
filter = vaultwarden
logpath = /logs/vaultwarden.log
action = iptables-allports[name=vaultwarden, chain=FORWARD]
#         %(action_mwl)s

[vaultwarden-admin]
enabled = true
port = 80,443
filter = vaultwarden-admin
logpath = /logs/vaultwarden.log
action = iptables-allports[name=vaultwarden-admin, chain=FORWARD]
#         %(action_mwl)s
```

### 3.4 启动Vaultwarden

```bash
root@instance-20210111-1623:~/bitwarden# docker-compose up -d
Creating network "vaultwarden_default" with the default driver
Creating vaultwarden ... done
Creating watchtower  ... done
Creating fail2ban    ... done
Creating caddy       ... done
```

查看一下各个容器的状态

```bash
root@instance-20210111-1623:~/bitwarden# docker-compose ps
   Name                  Command                  State                                         Ports
------------------------------------------------------------------------------------------------------------------------------
vaultwarden   /usr/bin/dumb-init -- /sta ...   Up (healthy)   3012/tcp, 80/tcp
caddy         caddy run --config /etc/ca ...   Up             2019/tcp, 0.0.0.0:3012->3012/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:80->80/tcp
fail2ban      /entrypoint.sh fail2ban-se ...   Up (healthy)
watchtower    /watchtower                      Up             8080/tcp
```

```bash
root@instance-20210111-1623:~/bitwarden# docker stats --no-stream
CONTAINER ID   NAME          CPU %     MEM USAGE / LIMIT   MEM %     NET I/O           BLOCK I/O         PIDS
ef510628c88b   caddy         0.00%     10.76MiB / 977MiB   1.10%     84.6kB / 89.5kB   5.46MB / 0B       8
6f9763592fe0   fail2ban      0.04%     14.75MiB / 977MiB   1.51%     0B / 0B           5.19MB / 12.3kB   7
4a378e7a8c98   watchtower    0.00%     5.016MiB / 977MiB   0.51%     1.48kB / 0B       5.51MB / 0B       7
344497bdf1aa   vaultwarden   0.00%     11.46MiB / 977MiB   1.17%     48.6kB / 28.9kB   5.31MB / 0B       28
```

### 3.5 绑定域名及检查fail2ban功能

将域名解析到服务器IP即可使用https访问Vaultwarden。可以使用各类型的解析服务，比如cloudflare、dnspod、he.net 等。

使用任意邮箱及密码登陆，会提示账号密码错误，同时在fail2ban日志中会记录该IP地址。

```bash
root@instance-20210111-1623:~/bitwarden# cat logs/fail2ban.log
2021-03-02 17:19:29,482 fail2ban.filter         [1]: INFO    [vaultwarden] Found 111.111.111.111 - 2021-03-04 17:19:29
```

如该IP地址为正在使用的IP地址，fail2ban功能正常。

注册账号后将Lastpass导出的密码库在web界面导入即可正常使用。

### 3.6 关闭注册及邀请注册

注册完成后可以修改 docker-compose.yml 内配置关闭用户注册及邀请注册，其配置参数如下

```bash
 environment:
      - SIGNUPS_ALLOWED=false        #关闭新用户注册
      - INVITATIONS_ALLOWED=false    #关闭组织邀请注册
```

其他相关environment请查看[官方文档](https://github.com/dani-garcia/vaultwarden/blob/main/.env.template)
## 四、备份及清除日志

### 4.1 备份 Vaultwarden 数据库

Vaultwarden 的模式决定了服务器端的数据永远是最新的，虽然 iOS 和 Android 的客户端支持导出数据，但客户端不会在后台自动同步密码库。若长时间未打开，密码库将不会是最新版本。

此时对服务器数据进行备份就显得尤为重要。可以定时将将容器挂载目录打包，上传到网络存储。

以通过 webdav 上传到坚果云为例。

新建配置文件

```bash
vim /etc/cron.daily/vaultwarden-backup
```

```shell
#!/bin/sh
set -e

BACKUP_DIR="/root/vaultwarden"  # 备份存放目录
BACKUP_PREFIX="vaultwarden-"  # 备份文件名前缀
USERNAME="USERNAME" # 坚果云用户名
APP_PASSWD="APP_PASSWD" # 坚果云应用密码
DAV_URL="https://dav.jianguoyun.com/dav/vaultwarden/" # 坚果云DAV地址,vaultwarden可自定义

filename="${BACKUP_PREFIX}$(date +%F).tar.gz"
cd "${BACKUP_DIR}"

# 创建备份
tar czf "${filename}" bw-data/

# 上传备份
curl -u "${USERNAME}:${APP_PASSWD}" -T "${filename}" "${DAV_URL}"

# 删除临时文件
rm "${BACKUP_DIR}/${filename}"
```

2025年3月12日新增一个备份脚本，可设置WEBDAV备份保存份数,自动删除，避免手动清理

```shell
#!/bin/sh
set -e

BACKUP_DIR="/root/vaultwarden"         # 备份存放目录
BACKUP_PREFIX="vaultwarden-"           # 备份文件名前缀
MAX_BACKUPS=10                       # WebDAV 上保留的最大备份数量
USERNAME="username"      # 坚果云用户名
APP_PASSWD="password"         # 坚果云应用密码
DAV_URL="https://dav.jianguoyun.com/dav/vaultwarden/"  # 坚果云DAV地址,vaultwarden可自定义

filename="${BACKUP_PREFIX}$(date +%F).tar.gz"
cd "${BACKUP_DIR}"

# 创建备份
tar czf "${filename}" bw-data/

# 上传备份
curl -u "${USERNAME}:${APP_PASSWD}" -T "${filename}" "${DAV_URL}"

# 获取 WebDAV 备份列表并按名称（日期）排序
webdav_response=$(curl -s -u "${USERNAME}:${APP_PASSWD}" -X PROPFIND --data '<?xml version="1.0" encoding="UTF-8"?>
<D:propfind xmlns:D="DAV:">
  <D:prop><D:getlastmodified/></D:prop>
</D:propfind>' -H "Depth: 1" "${DAV_URL}")

# 输出调试信息（可选）
# echo "WebDAV 响应内容："
# echo "${webdav_response}"

# 如果未获取到数据，则输出警告并跳过删除旧备份操作
if [ -z "$webdav_response" ]; then
  echo "Warning: 未能获取到 WebDAV 文件列表，跳过旧备份删除操作"
else
  backup_list=$(echo "${webdav_response}" | grep -ioP '(?<=<d:href>).*?(?=</d:href>)' | grep "${BACKUP_PREFIX}" | sed 's/.*\///' | sort)
  
  # 检查是否有获取到备份列表
  if [ -z "${backup_list}" ]; then
    echo "Warning: WebDAV 文件列表为空，可能未上传成功或格式不符，跳过删除操作"
  else
    backup_count=$(echo "${backup_list}" | wc -l)
    if [ "${backup_count}" -gt "${MAX_BACKUPS}" ]; then
      delete_count=$((backup_count - MAX_BACKUPS))
      echo "检测到 ${backup_count} 个备份，删除最旧的 ${delete_count} 个..."
      echo "${backup_list}" | head -n "${delete_count}" | while read -r old_backup; do
        echo "删除 ${old_backup} ..."
        curl -u "${USERNAME}:${APP_PASSWD}" -X DELETE "${DAV_URL}${old_backup}"
      done
    fi
  fi
fi

# 删除本地临时文件
rm -f "${BACKUP_DIR}/${filename}"

```

增加执行权限

```bash
chmod a+x /etc/cron.daily/vaultwarden-backup
```

坚果云上新建同步文件夹，名称为 `vaultwarden` 。运行 `vaultwarden-backup` 即可测试同步功能。

```shell
sh /etc/cron.daily/vaultwarden-backup
```

### 4.2 定期清空日志

创建脚本文件

```bash
vim /root/bitwarden/del_vaultwarden_log.sh
```

```bash
#! /bin/bash
cat /dev/null > /root/bitwarden/logs/vaultwarden.log # 清空Vaultwarden的日志记录
cat /dev/null > /root/bitwarden/logs/fail2ban.log # 清空fail2ban的日志记录
```

增加执行权限

```bash
chmod +x /root/bitwarden/del_vaultwarden_log.sh
```

创建自动执行任务：

```bash
crontab -e # 打开定时任务
0 2 * * 7 /root/bitwarden/del_vaultwarden_log.sh # 添加一行新任务：每周日凌晨2点自动执行此脚本
/etc/init.d/cron restart # 重启定时任务
```

## 五、参考

- [Configuration overview · dani-garcia/vaultwarden Wiki](https://github.com/dani-garcia/vaultwarden/wiki/Configuration-overview)
- [bitwarden_rs 部署和使用 - Bitwarden 部署和使用](https://host.bitwarden.in/deploying-and-using-of-bitwarden_rs)
- [Caddy Documentation](https://caddyserver.com/docs/)
