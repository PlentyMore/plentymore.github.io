---
title: sslibev
date: 2018-08-01 00:50:58
tags:
---
docker-compose.yml
```$xslt
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

run `docker-compose up -d`

{% meting "27455897" "netease" "playlist" "listmaxheight:340px" "theme:#ad7a86"%}