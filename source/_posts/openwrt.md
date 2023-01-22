---
title: 我的openwrt设置
date: 2023-01-18 19:15:00
tags:
- network
- proxy
- openwrt
- 网络
- 代理
---

# 我的openwrt设置

## 设备

倍控 J4125软路由2.5G网卡2500M爱快维盟I226电脑N5105四核虚拟机

G31黑N5105四网226-2.5G双内存M2

## 固件

esir 高大全

https://drive.google.com/file/d/1hrsWtbyzxrbmrxfOGfU80Si5ivXKBmRF/view?usp=sharing

## 上网

修改`/etc/config/network`，执行`service network reload`

```
config interface 'loopback'
        option proto 'static'
        option ipaddr '127.0.0.1'
        option netmask '255.0.0.0'
        option device 'lo'

config interface 'lan'
        option proto 'static'
        option ipaddr '192.168.5.1'
        option netmask '255.255.255.0'
        option type 'bridge'
        option ifname 'eth0 eth2 eth3'
        option delegate '0'
        option force_link '0'
        option gateway '192.168.5.1'

config interface 'wan'
        option device 'eth1'
        option proto 'pppoe'
        option username '宽带帐号'
        option password '宽带密码'
        option ifname 'eth1'
        option keepalive '0'
        option delegate '0'
        option ipv6 '0'

config interface 'ipsec_server'
        option ifname 'ipsec0'
        option device 'ipsec0'
        option proto 'static'
        option ipaddr '192.168.100.1'
        option netmask '255.255.255.0'

config interface 'docker'
        option ifname 'docker0'
        option proto 'none'
        option auto '0'

config device
        option type 'bridge'
        option name 'docker0'
```

## 防火墙

http://192.168.5.1/cgi-bin/luci/admin/network/firewall/zones

wan 到 lan，docker 禁止入站，转发

需要开的端口使用端口映射或通信规则

## 端口映射

http://192.168.5.1/cgi-bin/luci/admin/network/firewall/forwards

## 动态dns

http://192.168.5.1/cgi-bin/luci/admin/services/ddns

高级设置那里ip地址来源选择网络，wan

## 扩容

https://youtu.be/YwbwzuXKNlg

- `lsblk` 查看硬盘
- `cfdisk /dev/sda` 新建分区
- `mkfs.ext4 /dev/sda3` 格式化
- `cp /overlay/* /mnt/sda3/` 复制当前系统到新分区
- http://192.168.5.1/cgi-bin/luci/admin/system/fstab 挂载点下添加，分区sda3挂载到overlay
- 重启

docker同理，挂载到`/opt/docker`

## 备份

如果在没有重刷固件的情况下，仅对`/overlay`进行打包并备份

```bash
cd /tmp

tar -czvf /tmp/overlay_backup_$(date +%Y-%m-%d-%H-%M-%S).tar.gz /overlay

mv /tmp/overlay_backup_* /mnt/sda1/backup/openwrt/

```

然后下次直接将 overlay_backup.tar.gz 上传至`/tmp`，然后清空`/overlay`并恢复备份

```bash
cp /mnt/sda1/backup/openwrt/overlay_backup.tar.gz /tmp/

rm -rvf /overlay/*

cd / && tar -xzvf /tmp/overlay_backup.tar.gz

```

## openclash

基本用法不介绍了

如果有额外的简单自定义规则，可以使用全局设置->规则设置（访问控制）->自定义规则（访问控制）

遇到的问题：

域名不使用我配置的host解析：是因为这个域名根据规则需要代理，被发到代理服务器那里了，结果直接就去访问了真的服务器，然后把结果返回了

## alist

https://github.com/sbwml/luci-app-alist

How to install prebuilt packages (OpenWrt 21,22,master)

- Login OpenWrt terminal (SSH)
- Install `curl` package
  ```shell
  opkg update
  opkg install curl
  ```
- Execute install script (Multi-architecture support)
  ```shell
  sh -c "$(curl -ksS https://raw.githubusercontent.com/sbwml/luci-app-alist/master/install.sh)"
  ```

## emby

docker

```yaml
version: "2.3"
services:
  emby:
    image: emby/embyserver
    container_name: embyserver
    network_mode: host # Enable DLNA and Wake-on-Lan
    environment:
      - UID=0 # The UID to run emby as root (default: 2)
      - GID=0 # The GID to run emby as root (default 2)
      - GIDLIST=0 # A comma-separated list of additional GIDs to run emby as root (default: 2)
    volumes:
      - /root/workspace/emby/config:/config # Configuration directory
      - /mnt/sda1:/mnt/sda1 # Media directory
    ports:
      - 8096:8096 # HTTP port
      - 8920:8920 # HTTPS port
    devices:
      - /dev/dri:/dev/dri # VAAPI/NVDEC/NVENC render nodes
    restart: unless-stopped
```

破解

https://yuanfangblog.xyz/technology/159.html

主要思路是部署一个伪装成mb3admin.com的https服务，本地安装自己的ca根证书，用这个根证书签一个mb3admin.com的证书，验证时返回成功

```
    location /admin/service/registration/validateDevice {
        default_type "application/json";
        return 200 '{"cacheExpirationDays": 3650,"message": "Device Valid (limit not checked)","resultCode": "GOOD"}';
    }

    location /admin/service/registration/validate {
        default_type "application/json";
        return 200 '{"featId": "","registered": true,"expDate": "2099-01-01","key": ""}';
    }
        

    location /admin/service/registration/getStatus {
        default_type "application/json";
        return 200 '{planType: "Lifetime", deviceStatus: 0, subscriptions: []}';
    }

    location /admin/service/appstore/register {
        default_type "application/json";
        return 200 '{"featId": "","registered": true,"expDate": "2099-01-01","key": ""}';
    }

    location /emby/Plugins/SecurityInfo {
        default_type "application/json";
        return 200 '{SupporterKey: "", IsMBSupporter: true}';
    }
```

注意

如果之前在本地其他电脑上部署过emby且启用了upnp，可能会导致端口冲突

## 根证书

openwrt 所信任的根证书位于`/etc/ssl/cert.pem`，只要修改这个文件，把你自建的ca证书加到后面即可。

## qbittorrent

### rss 自动下载

https://github.com/MisakaFxxk/MisakaF_Emby/tree/main/tvshows/anime

### 添加tracker

https://github.com/Jorman/Scripts#addqbittorrenttrackerssh

## samba 共享

win10无法连接？

https://blog.csdn.net/iamlihongwei/article/details/79377657


可以右键挂载为网络驱动器

网络驱动器我以Z盘开始，往前推

存在问题：

输密码无法连接

## 端口规划

有默认的，就用默认的，毕竟他们已经试过了，也符合大众印象

冲突了，就从10000开始+1

在这里我记录一下

- 10000 openwrt web ui
- ~~10001 vs code server~~
- 10002 qbittorrent

## code-server

### linuxserver/code-server

```yaml
version: "2.1"
services:
  code-server:
    image: lscr.io/linuxserver/code-server:latest
    container_name: code-server
    environment:
      - PUID=0
      - PGID=0
      - TZ=Asia/ShangHai
      - PASSWORD=密码 #optional
      - HASHED_PASSWORD= #optional
      - SUDO_PASSWORD=密码 #optional
      - SUDO_PASSWORD_HASH= #optional
      - PROXY_DOMAIN=code.awnlzw.com #optional
      - DEFAULT_WORKSPACE=/root/workspace #optional
      - DOCKER_USER=root
    volumes:
      - /root/workspace/code-server/config:/config
      - /root/workspace:/root/workspace
      - /root/.ssh:/root/.ssh
    ports:
      - 8443:8443
    restart: unless-stopped
```

### codercom/code-server

```bash
mkdir -p ~/.config

docker run -it --name code-server -p 10001:8080 \
  -v "/root/workspace/code-server/config/.config:/root/.config" \
  -v "/root/workspace:/root/workspace" \
  -v "/root/workspace/code-server/config/data:/root/.local/share/code-server" \
  -v "/root/workspace/code-server/config/extensions:/root/.local/share/code-server/extensions" \
  -v "/root/.ssh:/root/.ssh" \
  -u "$(id -u):$(id -g)" \
  -e "DOCKER_USER=$USER" \
  codercom/code-server:latest

```

启动后的操作

安装中文插件

安装常用工具

```bash
#! /bin/bash

apt update

apt upgrade -y

apt install zsh

sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

chsh -s $(which zsh)

apt install python3 -y

update-alternatives --install /usr/bin/python python /usr/bin/python3 100

apt install python3-pip -y

pip install requests

git config --global user.name awnlzw

git config --global user.email awnlzw@awnlzw.com

git clone git@github.com:ICE-GB/blog.git

git clone --recurse-submodules git@github.com:ICE-GB/tools.git
```
## kasm

> Kasm Workspaces是一个docker容器流平台，用于提供基于浏览器的桌面、应用程序和网络服务访问

Ubuntu22.04桌面

用户名为`kasm_user`

```bash
sudo docker run --rm -it --shm-size=512m -p 6901:6901 -e VNC_PW=密码 kasmweb/ubuntu-jammy-desktop:1.12.0
```

```yaml
version: "2.3"
services:
  ubuntu:
    image: kasmweb/ubuntu-jammy-desktop:1.12.0
    container_name: ubuntu
    shm_size: 512m
    environment:
      - VNC_PW=密码 
    volumes:
      - /mnt/sda1:/mnt/sda1
    ports:
      - 6901:6901 # HTTPS port
    devices:
      - /dev/dri:/dev/dri # VAAPI/NVDEC/NVENC render nodes
    restart: unless-stopped
```

注意：

使用nginx反代时，需要修改设置->高级->中继站ID->websocket为正确的路径

## 外网回家

### zerotier

https://blog.minws.com/archives/809/

- 登录zerotier，创建networt
- http://192.168.5.1/cgi-bin/luci/admin/vpn/zerotier/general 启用zerotier，输入networkid，自动允许客户端NAT
- 在zerotier后台，Members 成员那边对openwrt勾选Auth
- 在zerotier后台，Managed Routes 路由管理那边添加路由。左边为内网lan网段，右边为openwrt在zerotier中的ip
- 客户端连接network，在zerotier后台，Members 成员那边对客户端勾选Auth

```
192.168.5.0/24  via  192.168.196.153
```

#### 自建moon节点

https://www.wnark.com/archives/152.html

https://www.v2ex.com/t/768628

## nginx

配置文件在`/etc/nginx/uci.conf`

测试配置文件是否正确`nginx -t -c /etc/nginx/uci.conf`

测试详情`nginx -T -c /etc/nginx/uci.conf`

它引入了`/etc/nginx/conf.d/*.conf`作为server，`/etc/nginx/conf.d/*.location`作为location

### 使用nginx来访问原本的管理页面

修改uhttpd的监听端口为10000

新建`/etc/nginx/conf.d/openwrt.conf`来代理到`http://127.0.0.1:10000`

```
server {
    listen       443 ssl;
    
    server_name openwrt.awnlzw.com;

    ssl_certificate    /etc/nginx/conf.d/awnlzw.com.pem;
    ssl_certificate_key    /etc/nginx/conf.d/awnlzw.com.key;

    access_log /var/log/nginx/openwrt_access.log;
    error_log /var/log/nginx/openwrt_error.log;    

    location /alist/ {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host "alist.awnlzw.com";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Range $http_range;
        proxy_set_header If-Range $http_if_range;
        proxy_redirect off;
        proxy_pass https://alist.awnlzw.com:443/;
        # the max size of file to upload
        client_max_body_size 20000m;
    }

    location /clash/ {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host "clash.awnlzw.com";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Range $http_range;
        proxy_set_header If-Range $http_if_range;
        proxy_redirect off;
        proxy_pass https://clash.awnlzw.com:443/;
        # the max size of file to upload
        client_max_body_size 20000m;
    }

    location /code/ {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host "code.awnlzw.com";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Range $http_range;
        proxy_set_header If-Range $http_if_range;
        proxy_redirect off;
        proxy_pass https://code.awnlzw.com:443/;
        # the max size of file to upload
        client_max_body_size 20000m;
    }

    location /emby/ {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host "emby.awnlzw.com";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Range $http_range;
        proxy_set_header If-Range $http_if_range;
        proxy_redirect off;
        proxy_pass https://emby.awnlzw.com:443/;
        # the max size of file to upload
        client_max_body_size 20000m;
    }

    location /portainer/ {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host "portainer.awnlzw.com";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Range $http_range;
        proxy_set_header If-Range $http_if_range;
        proxy_redirect off;
        proxy_pass https://portainer.awnlzw.com:443/;
        # the max size of file to upload
        client_max_body_size 20000m;
    }

    location /qbittorrent/ {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host "qbittorrent.awnlzw.com";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Range $http_range;
        proxy_set_header If-Range $http_if_range;
        proxy_redirect off;
        proxy_pass https://qbittorrent.awnlzw.com:443/;
        # the max size of file to upload
        client_max_body_size 20000m;
    }

    location /ubuntu/ {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host "ubuntu.awnlzw.com";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Range $http_range;
        proxy_set_header If-Range $http_if_range;
        proxy_redirect off;
        proxy_pass https://ubuntu.awnlzw.com:443/;
        # the max size of file to upload
        client_max_body_size 20000m;
    }
    
    location / {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Range $http_range;
        proxy_set_header If-Range $http_if_range;
        proxy_redirect off;
        proxy_pass http://192.168.5.1:10000;
        # the max size of file to upload
        client_max_body_size 20000m;
    }
}
```

### nginx 代理alist

由于加了一个路径，alist那边的config.json中需要配置site_url为`https://openwrt.awnlzw.com/alist`

我安装的alist服务它默认没这个配置，需要添加

`config_get site_url $1 site_url "https://openwrt.awnlzw.com/alist"`

新建配置文件时把site_url写入进去

`cat > $CONFIG_DIR/config.json <<EOF
{"force":false,"address":"$listen_addr","port":$port,"site_url":"$site_url","cdn":"","jwt_secret":"","token_expires_in":$token_expires_in,"database":{"type":"sqlite3","host":"","port":0,"user":"","password":"","name":"","db_file":"/etc/alist/data.db","table_prefix":"x_","ssl_mode":""},"scheme":{"https":$SSL,"cert_file":"$ssl_cert","key_file":"$ssl_key"},"temp_dir":"$temp_dir","log":{"enable":$LOG,"name":"$temp_dir/alist.log","max_size":10,"max_backups":5,"max_age":28,"compress":false}}
EOF`

```
#!/bin/sh /etc/rc.common

START=99
USE_PROCD=1
PROG=/usr/bin/alist
CONFIG_DIR=/etc/alist

get_config() {
	config_get_bool enabled $1 enabled 1
	config_get port $1 port 5244
	config_get site_url $1 site_url "https://openwrt.awnlzw.com/alist"
	config_get log $1 log 1
	config_get temp_dir $1 temp_dir "/tmp/alist"
	config_get ssl $1 ssl 0
	config_get ssl_cert $1 ssl_cert ""
	config_get ssl_key $1 ssl_key ""
	config_get token_expires_in $1 token_expires_in 48
	config_get allow_wan $1 allow_wan 0
	config_load network
	config_get lan_addr lan ipaddr "0.0.0.0"
}

set_firewall() {
	if [ "$external_access" = "allow" ]; then
		uci -q delete firewall.alist
		uci set firewall.alist=rule
		uci set firewall.alist.name="alist"
		uci set firewall.alist.target="ACCEPT"
		uci set firewall.alist.src="wan"
		uci set firewall.alist.proto="tcp"
		uci set firewall.alist.dest_port="$port"
		uci set firewall.alist.enabled="1"
		uci commit firewall
		/etc/init.d/firewall reload >/dev/null 2>&1
	elif [ "$external_access" = "deny" ]; then
		uci -q delete firewall.alist
		uci commit firewall
		/etc/init.d/firewall reload >/dev/null 2>&1
	fi
}

start_service() {
	config_load alist
	config_foreach get_config alist
	[ $enabled != 1 ] && return 1
	mkdir -p $temp_dir
	[ "$ssl" -eq 1 ] && SSL=true || SSL=false
	[ "$log" -eq 1 ] && LOG=true || LOG=false
	if [ "$allow_wan" -eq "1" ]; then
		listen_addr="0.0.0.0"
		external_access="allow"
	else
		listen_addr=$lan_addr
		external_access="deny"
	fi
	set_firewall
	cat /dev/null > $temp_dir/alist.log
	cat > $CONFIG_DIR/config.json <<EOF
{"force":false,"address":"$listen_addr","port":$port,"site_url":"$site_url","cdn":"","jwt_secret":"","token_expires_in":$token_expires_in,"database":{"type":"sqlite3","host":"","port":0,"user":"","password":"","name":"","db_file":"/etc/alist/data.db","table_prefix":"x_","ssl_mode":""},"scheme":{"https":$SSL,"cert_file":"$ssl_cert","key_file":"$ssl_key"},"temp_dir":"$temp_dir","log":{"enable":$LOG,"name":"$temp_dir/alist.log","max_size":10,"max_backups":5,"max_age":28,"compress":false}}
EOF
	procd_open_instance alist
	procd_set_param command $PROG
	procd_append_param command server --data $CONFIG_DIR
	procd_set_param stdout 1
	procd_set_param stderr 1
	procd_set_param respawn
	procd_close_instance alist
}

service_triggers() {
	procd_add_reload_trigger "alist"
}

stop_service() {
	external_access="deny"
	set_firewall
}

reload_service() {
	stop
	start
}

```

### 信任cloudflare的证书

由于用的是cloudflare签的证书，需要在浏览器所在的电脑上安装cloudflare的ca根证书

https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/#:~:text=Some%20origin%20web%20servers%20require%20upload%20of%20the%20Cloudflare%20Origin%20CA%20root%20certificate.%20Click%20a%20link%20below%20to%20download%20either%20an%20RSA%20and%20ECC%20version%20of%20the%20Cloudflare%20Origin%20CA%20root%20certificate%3A