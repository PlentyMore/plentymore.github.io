---
title: 使用VPS下载bt资源
date: 2019-01-11 23:06:35
tags:
    - Misc
---

迅雷无法下载一些bt资源，而且迅雷没有Linux版，用别的bt下载软件来下载的话因为没有公网ip所以速度比较尴尬，刚好手上有几台vps，有公网ip，而且每个月剩下900多G流量用过不完，刚好可以用来下载这些资源。

## 安装docker

docker的安装可以参考我的[另一篇博客](https://plentymore.github.io/2018/08/01/sslibev/#%E5%AE%89%E8%A3%85docker)

## 编写docker-cocmpose.yml

文件我已经写好了，根据自己的需要修改里面的参数就行。文件放在了[github项目](https://github.com/PlentyMore/vps-deluge-nginx.git)上

### 克隆项目

首先登录上vps，把项目克隆下来，然后进入克隆的项目目录。

```bash
git clone https://github.com/PlentyMore/vps-deluge-nginx.git
cd vps-deluge-nginx/
```

### 修改deluge和nginx配置

打开`docker-compose.yml`文件

```bash
version: "3"
services:
  deluge:
    image: linuxserver/deluge
    container_name: deluge
    network_mode: host
    environment:
      - PUID=0 # 需要修改
      - PGID=0 # 需要修改
      - TZ=Europe/London # 改不改都行
    volumes:
      - ./deluge:/config
      - ./downloads:/root/Downloads
    restart: unless-stopped
  nginx:
    image: nginx
    container_name: nginx
    volumes:
    - ./downloads:/usr/share/nginx/html
    - ./nginx:/etc/nginx/conf.d
    ports:
    - "80:80" # 可以修改前面的端口
    restart: unless-stopped

```

需要修改的地方只有下面这个地方
```bash
      - PUID=你的UID
      - PGID=你的GID
```

在命令行输入id，会得到下面的结果：
```bash
uid=0(root) gid=0(root) groups=0(root)
```
根据得到的结果修改就行了。还可以根据需要改很多东西，可以参考[文档](https://hub.docker.com/r/linuxserver/deluge)


修改完`docker-compose.yml`文件后，还需要修改`default.conf`文件，该文件在`vps-deluge-nginx/nginx`目录下
```bash
server {
    listen  80;
    server_name    2.3.3.3; # 替换成你的vps公网ip
    charset utf-8; # 避免中文乱码
    root /usr/share/nginx/html;
    location / {
        autoindex on; # 索引
        autoindex_exact_size on; # 显示文件大小
        autoindex_localtime on; # 显示文件时间
    }
}
```
只需要修改server_name后面的ip，改成你的vps的公网ip就可以了。


## 运行

在`vps-deluge-nginx`目录输入如下命令
```bash
docker-compose up -d
```
然后就可以使用了。

### 使用deluge
在浏览器输入http://你的ip:8112就可以访问了，第一次访问需要输入密码，默认密码为deluge，最好修改一下密码，当然登录后它会提示你修改密码。

![Imgur](https://i.imgur.com/OYpziJH.png)

登入后，提示修改密码，选择yes

![Imgur](https://i.imgur.com/3OI4ci9.png)

输入旧密码和新密码，点击change

![Imgur](https://i.imgur.com/WAUOS0r.png)

修改完成后选择Connection Manage

![Imgur](https://i.imgur.com/5dHV9wd.png)

然后选中里面唯一一个online状态的连接，然后点击connect

![Imgur](https://i.imgur.com/AIjtx7y.png)

连接之后就可以添加bt任务下载资源了，可以从本地选择种子，也可以提供链接

![Imgur](https://i.imgur.com/RRxYfw2.png)

添加完任务后，就会自动下载了

![Imgur](https://i.imgur.com/T6i4YXT.png)

### 使用nginx
在浏览器输入http://你的ip，如果你改了nginx的端口，则需要加上端口

可以直接看到下载的资源，点击资源还可以在线观看，也可以复制地址栏的链接粘贴到下载器下载

![Imgur](https://i.imgur.com/13BixW3.png)

![Imgur](https://i.imgur.com/xfdah6r.png)

### 关闭
在`vps-deluge-nginx`目录下面输入以下命令：
```
docker-compose stop
```
这样就可以关闭deluge和nginx了。

如果需要彻底移除，可以输入如下命令：
```
docker-compose down
```
然后把`vps-deluge-nginx`目录整个删除




