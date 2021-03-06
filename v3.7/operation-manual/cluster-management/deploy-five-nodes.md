---
title: 五节点高可用部署方案
summary: 介绍五节点的高可用部署方案，达到第二级高可用
toc: true
toc_not_nested: true
asciicast: true
---

<div class="filters filters-big clearfix">
    <a href="five-nodes-deployment.html"><button class="filter-button ">方案概述</button></a>
    <a href="deploy-five-nodes.html"><button class="filter-button current"><strong>操作流程</strong></button></a>
</div>

<div id="toc"></div>

##一、基本平台部署

- 按正常顺序部署管理节点3个、计算节点2个

```bash
#in the first manage node 
wget https://pkg.rainbond.com/releases/common/v3.7.0/grctl
chmod +x ./grctl
./grctl init

#add second manage node
grctl node add --hostname manage02 --iip <内网ip> --root-pass <root用户密码> --role manage

#add 3st manage node 
grctl node add --hostname manage03 --iip <内网ip> --root-pass <root用户密码> --role manage

#add the first compute node
grctl node add --hostname compute01 --iip <内网ip> --root-pass <root用户密码> --role compute

#add the second compute node
grctl node add --hostname compute02 --iip <内网ip> --root-pass <root用户密码> --role compute

```
## 二、部署Glusterfs

在计算节点部署双节点GFS集群：

- 安装GFS：

  - 详情参见：[GlusterFS安装]( https://www.rainbond.com/docs/stable/operation-manual/storage/GlusterFS/install.html)

> 注意：将计算节点作为存储节点，需要将上方文档中的 server1、server2 更换为 compute01、compute02

- 切换存储

为管理节点安装GFS文件系统

```bash
yum install -y centos-release-gluster
yum install -y glusterfs-fuse
```
将管理节点的/grdata目录写入GFS

```bash
mount -t glusterfs compute01:data /mnt
cp -rp /grdata/* /mnt
umount /mnt
```
编辑所有节点的/etc/fstab,新增一行：

manage01-03 & compute01:

```bash
compute01:/data	/grdata	glusterfs	backupvolfile-server=compute02,use-readdirp=no,log-level=WARNING,log-file=/var/log/gluster.log 0 0
```

compute02:

```bash
compute02:/data	/grdata	glusterfs	backupvolfile-server=compute01,use-readdirp=no,log-level=WARNING,log-file=/var/log/gluster.log 0 0
```

重新挂载

```bash
mount -a
```
## 三、配置VIP

在两个计算节点配置VIP，搭建基于keepalived软件的主备机制

- 安装keepalived

```bash
yum install -y keepalived
```


- 修改配置文件

```bash
#备份原有配置文件
cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
#编辑配置文件如下
#compute01
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state MASTER   #角色，当前为主节点
    interface ens6f0  #网卡设备名，通过 ifconfig 命令确定
    virtual_router_id 51
    priority 100   #优先级，主节点大于备节点
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        <VIP>
    }
}

#compute02
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state BACKUP   #角色，当前为备节点
    interface ens6f0   #网卡设备名，通过 ifconfig 命令确定
    virtual_router_id 51
    priority 50   #优先级，主节点大于备节点
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        <VIP>
    }
}


#启动服务，设置开机自启动
systemctl start keeplived
systemctl enable keeplived
```

- 切换应用域名解析IP到VIP

```bash
#在manage01节点执行
grctl domain --ip <VIP>
```

- 修改tcpdomain

```bash
#在manage01节点执行
din rbd-db   #进入rbd-db组件
mysql        #进入数据库
use console; #使用console数据库
UPDATE region_info set tcpdomain="<VIP>"; #更新tcpdomain
```

## 四、配置本地主机解析

- 修改5个节点的/etc/hosts

  ```bash
  vi /etc/hosts
  #修改如下
  <manage01 IP> manage01 <host_UUID> kubeapi.goodrain.me region.goodrain.me console.goodrain.me
  
  <VIP> goodrain.me lang.goodrain.me maven.goodrain.me
  ```

## 五、添加rbd-lb配置

- 修改compute01-02的rbd-lb组件配置

```bash
vi /opt/rainbond/etc/rbd-lb/dynamics/dynamic_servers/default.http.conf
#修改如下

upstream lang {
    server <manage01_IP>:8081;
    server <manage02_IP>:8081;
    server <manage03_IP>:8081;
}

upstream maven {
    server <manage01_IP>:8081;
    server <manage02_IP>:8081;
    server <manage03_IP>:8081;
}

upstream registry {
    ip_hash;
    server <manage01_IP>:5000;
    server <manage02_IP>:5000;
    server <manage03_IP>:5000;
}
upstream rbd-api {
    server <manage01_IP>:6060;
    server <manage02_IP>:6060;
    server <manage03_IP>:6060;
}

server {
    listen 6060;
    proxy_pass rbd-api;
    proxy_connect_timeout 60;
    proxy_read_timeout 600;
    proxy_send_timeout 600;
}

server {
    listen 80;
    server_name lang.goodrain.me;
    rewrite ^/(.*)$ /artifactory/pkg_lang/$1 break;
    location / {
        proxy_pass http://lang;
        proxy_set_header Host $host;
        proxy_redirect off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 60;
        proxy_read_timeout 600;
        proxy_send_timeout 600;
    }
}

server {
    listen 80;
    server_name maven.goodrain.me;
    location / {
        rewrite ^/(.*)$ /artifactory/libs-release/$1 break;
        proxy_pass http://maven;
        proxy_set_header Host $host;
        proxy_redirect off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 60;
        proxy_read_timeout 600;
        proxy_send_timeout 600;
    }

    location /monitor {
        return 204;
    }
}

server {
    listen 443;
    server_name goodrain.me;
    ssl on;
    ssl_certificate /usr/local/openresty/nginx/conf/dynamics/dynamic_certs/goodrain.me/server.crt;
    ssl_certificate_key /usr/local/openresty/nginx/conf/dynamics/dynamic_certs/goodrain.me/server.key;
    client_max_body_size 0;
    chunked_transfer_encoding on;
    location /v2/ {
        proxy_pass http://registry;
        proxy_set_header Host $http_host; # required for docker client's sake
        proxy_set_header X-Real-IP $remote_addr; # pass on real client's IP
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 900;
    }
}
# repo.goodrain.me
server {
    listen 80;
    root /opt/rainbond/install/install/pkgs/centos/;
    server_name repo.goodrain.me;

}

```

## 六、配置Cockroachdb

受限于docker化的分布式数据库Cockroachdb并不稳定，当前部署方案需要内测一段时间才会继续更新。