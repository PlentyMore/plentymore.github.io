---
title: HTTP消息格式
date: 2019-01-12 15:08:33
tags:
    - HTTP
---

这篇博客的主要目的是对[HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol)的消息格式（主要是请求端和响应端的格式）进行总结。

## Request

请求端的消息格式如下：

* __Request line__, such as `GET /logo.gif HTTP/1.1`

* __Headers__ 比如`Host : www.google.com`（从HTTP/1.1版本开始必须有Host这个消息头，其它的消息头是可选的，具体有哪些标准请求头可查看[Request_fields](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields#Request_fields)）

* __An empty line__ `\r\n`，用来把__Headers__和__HTTP message body__隔开

* __Optional HTTP message body data__ 这部分是要发给服务端的[内容](https://en.wikipedia.org/wiki/HTTP_message_body)，是可选的

需要注意的是 __Request line__ 和每一个 __header__ 后面都要有`\r\n`作为结束标志，具体可查看[Request_message](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol#Request_message)

客户端请求示例：
```bash
GET /index.html HTTP/1.1
Host: www.example.com
```

## Response

响应端的消息格式如下：

* __Status line__ , which includes the status code and reason message (e.g., HTTP/1.1 200 OK, which indicates that the client's request succeeded.)

* __Response header fields__ (e.g., Content-Type: text/html)，具体有哪些标准响应头可查看[Response_fields](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields#Response_fields)，其它常用的非标准请求头也在同一个页面

* __An empty line__ `\r\n`，用来把__Response header fields__和__HTTP message body__隔开

* __Optional HTTP message body data__ 要返回给客户端的[内容](https://en.wikipedia.org/wiki/HTTP_message_body)，是可选的，比如返回一个HTML页面

和__Request__类似，__Status line__ 和每一个 __Response header fields__ 后面都要有`\r\n`作为结束标志,具体可查看[Response_message](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol#Response_message)

服务端相应消息实例：
```bash
HTTP/1.1 200 OK
Date: Mon, 23 May 2005 22:38:34 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 138
Last-Modified: Wed, 08 Jan 2003 23:11:55 GMT
Server: Apache/1.3.3.7 (Unix) (Red-Hat/Linux)
ETag: "3f80f-1b6-3e1cb03b"
Accept-Ranges: bytes
Connection: close

<html>
<head>
  <title>An Example Page</title>
</head>
<body>
  Hello World, this is a very simple HTML document.
</body>
</html>
```

## HTTP版本

目前HTTP的版本主要是HTTP/1.0，HTTP/1.1和HTTP/2。

在HTTP/1.0版本中，每请求一个资源（图片，视频，css文件等）都会发起一个新的请求，也就是和建立一条新的TCP连接，而在HTTP/1.1版中，请求资源的时候可以复用连接，不需要不断的发起新的请求，比如在请求完一个css文件后，再请求一个js文件，这时候可以继续使用请求css文件的时候建立的连接进行请求，不需要建立新的TCP连接进行请求，避免了TCP建立连接的三次握手开销。

HTTP/2增加了多路复用机制，具体可以查看[HTTP/2](https://en.wikipedia.org/wiki/HTTP/2)