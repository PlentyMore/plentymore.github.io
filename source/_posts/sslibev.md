---
title: sslibev
date: 2018-08-01 00:50:58
tags:
    - Misc
---

## 启用BBR加速
建议使用Ubuntu 16.04，因为BBR需要Linux kernel 4.9+ ，而且Ubuntu 16.04升级内核非常简单，Ubuntu 16.04启用BBR的流程如下：
1. 运行`apt install --install-recommends linux-generic-hwe-16.04`
安装期间会弹出几个选项让你选择，选择第一个即可。
2. 安装完成后，运行 `apt autoremove`移除多余的软件包（可选）
3. 运行`reboot`重启机器
4. 重启完毕后，运行`uname -r`查看内核版本，版本大于等于4.9就可以了
5. 运行`lsmod | grep bbr`,输出结果没有bbr字样的就继续执行下面的操作:<br>
```bash
modprobe tcp_bbr
echo "tcp_bbr" >> /etc/modules-load.d/modules.conf
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
```
接着运行下面两个命令：
```bash
sysctl net.ipv4.tcp_available_congestion_control
sysctl net.ipv4.tcp_congestion_control
```
输出结果有bbr，说明已经成功开启bbr。最后运行`lsmod | grep bbr`，输出结果有tcp_bbr，说明
bbr已经成功启动

## 优化server
将以下内容加入到/etc/sysctl.conf文件中
```properties
# max open files
fs.file-max = 51200
# max read buffer
net.core.rmem_max = 67108864
# max write buffer
net.core.wmem_max = 67108864
# default read buffer
net.core.rmem_default = 65536
# default write buffer
net.core.wmem_default = 65536
# max processor input queue
net.core.netdev_max_backlog = 4096
# max backlog
net.core.somaxconn = 4096

# resist SYN flood attacks
net.ipv4.tcp_syncookies = 1
# reuse timewait sockets when safe
net.ipv4.tcp_tw_reuse = 1
# turn off fast timewait sockets recycling
# net.ipv4.tcp_tw_recycle = 0
# short FIN timeout
net.ipv4.tcp_fin_timeout = 30
# short keepalive time
net.ipv4.tcp_keepalive_time = 1200
# outbound port range
net.ipv4.ip_local_port_range = 10000 65000
# max SYN backlog
net.ipv4.tcp_max_syn_backlog = 4096
# max timewait sockets held by system simultaneously
net.ipv4.tcp_max_tw_buckets = 5000
# turn on TCP Fast Open on both client and server side
net.ipv4.tcp_fastopen = 3
# TCP receive buffer
net.ipv4.tcp_rmem = 4096 87380 67108864
# TCP write buffer
net.ipv4.tcp_wmem = 4096 65536 67108864
# turn on path MTU discovery
net.ipv4.tcp_mtu_probing = 1
```
保存后，运行`sysctl -p`

## 安装docker
可以直接看[官方文档](https://docs.docker.com/install/linux/docker-ce/ubuntu/)<br>
这里我将官方文档的教程命令CV过来了方便操作。（下面的命令是Ubuntu16.04及以上版本的）
```bash
sudo apt-get remove docker docker-engine docker.io
sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install docker-ce
```
装好之后运行`sudo docker run hello-world`测试是否能正常运行。

## 安装docker-compose
可以直接看[官方文档](https://docs.docker.com/compose/install/)<br>
同样的我也将命令CV过来了。
```bash
sudo curl -L https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
然后运行`docker-compose --version`检查打印出的版本号。

## 编写docker-compose.yml文件
随便新建一个目录，比如`mkdir server`，然后进入该目录<br>
然后`vi docker-compose.yml`把下面的内容CV过去，根据需要修改就行了
```dockerfile
version: "3"
services:
  sslibev:
    image: shadowsocks/shadowsocks-libev
    environment:
      - PASSWORD=pwd
      - METHOD=rc4-md5
      - SERVER_PORT=2333
    restart: always
    ports:
      - "2333:2333"
      - "2333:2333/udp"
```
ports那里，`"2333:2333"`前面的2333可以修改为你客户端要链接的端口，比如443，后面的端口不需要改，改了也没什么意义

要改后面的端口的话同时也需要修改`- SERVER_PORT=2333`这里的端口，改成一样的就行，端口改太小会提示没有权限。

改前面的端口，需要同时修改两行，比如改成443,就要这样子：
```dockerfile
ports:
      - "443:2333"
      - "443:2333/udp"
```
保存文件后，运行`docker-compose up -d`即可启动服务端

{% meting "27455897" "netease" "playlist" "listmaxheight:340px" "theme:#ad7a86"%}