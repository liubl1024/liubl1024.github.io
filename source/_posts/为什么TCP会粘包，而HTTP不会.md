---
title: 为什么TCP会粘包，而HTTP不会
date: 2021-04-10 19:53:34
tags:
 - TCP
 - HTTP
 - 网络
categories:
    - 网络
---
### 为什么TCP 会粘包，而HTTP 不会

#### 什么是粘包

​	  粘包是在 客户端和服务端收发数据过程中，发生数据包粘连在一块无法区分，下面来看个例子


TCP 服务端使用 PHP 监听 8000 端口

```php
<?php

$ip = "127.0.0.1";
$port = "8000";
$recv_len = 24;

$serv = socket_create(AF_INET, SOCK_STREAM, SOL_TCP) or exit(socket_last_error()); //创建

socket_bind($serv, $ip, $port);
socket_listen($serv,4);
while(true){
    if ($cli = socket_accept($serv)) {
        while($msg = socket_read($cli,$recv_len)){
            echo "收到 数据: {$msg}\n";
        }
        socket_getsockname($cli, $cli_addr, $cli_port);
        $send_msg = "[发送数据]";
        socket_write($cli, $send_msg, strlen($send_msg));
        socket_close($cli);
    } else {
        usleep(1000);
    }
}
socket_close($serv);
```



client 使用 go 发送数据

```go
package main
import "net"
import "fmt"
import "time"

func main() {
	data := []byte("[我爱中国]")
	conn, err := net.DialTimeout("tcp", "127.0.0.1:8000", time.Second*30)
	if err != nil {
		fmt.Printf("连接失败, err : %v\n", err.Error())
        return
	}
	for i := 0; i <10; i++ {
		_, err = conn.Write(data)
		if err != nil {
			fmt.Printf("写入失败, err : %v\n", err)
			break
		}
	}
}
```



```
# php tcp_server.php
收到 数据: [我爱中国][我爱中
收到 数据: 国][我爱中国][我�
收到 数据: �中国][我爱中国][�
收到 数据: ��爱中国][我爱中�
收到 数据: �][我爱中国][我爱�
收到 数据: ��国][我爱中国]
```

可以看到 数据包粘连在一块无法区分的现象。形成这种现象的原因可以先看TCP 的定义

TCP/IP (Transmission Control Protocol/Internet Protocol) 即传输控制协议/网间协议，是一种面向连接的、可靠的、基于字节流的传输层（Transport layer）通信协议。

TCP （Transmission Control Protocol）协议即传输控制协议，根据  OSI 网络分层模型

![OSI七层模型| KEEM--BLOG](https://cdn.jsdelivr.net/gh/liubl1024/blog_images/image/blog/o1.png)

TCP 是四层即传输层协议，TCP 层，通过包序列号，确认应答等方式保证数据传输的可靠，TCP 只保证数据的传输可靠，但是对于同时发送多条数据来说 对TCP 而已只是一条数据流，TCP 层本身是不会去区分数据边界，数据边界是要靠上层协议，即应用层区分。



#### HTTP 协议为什么没有粘包问题

HTTP 协议是应用层协议，我们用 curl 开看 HTTP 的请求返回，以访问百度为例

```bash
$ curl -v http://www.baidu.com
*   Trying 220.181.38.150...
* TCP_NODELAY set
* Connected to www.baidu.com (220.181.38.150) port 80 (#0)
> GET / HTTP/1.1
> Host: www.baidu.com
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Accept-Ranges: bytes
< Cache-Control: private, no-cache, no-store, proxy-revalidate, no-transform
< Connection: keep-alive
< Content-Length: 2381
< Content-Type: text/html
< Date: Fri, 09 Apr 2021 16:26:59 GMT
< Etag: "588604c8-94d"
< Last-Modified: Mon, 23 Jan 2017 13:27:36 GMT
< Pragma: no-cache
< Server: bfe/1.0.8.18
< Set-Cookie: BDORZ=27315; max-age=86400; domain=.baidu.com; path=/
<
<!DOCTYPE html>
<!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=http://s1.bdstatic.com/r/www/cache/bdorz/baidu.min.css><title>百度一下，你就知道</title></head>
```

根据HTTP 协议定义，请求和响应都分为  `http head` 和 `http body` ，http head 和 body 通过 `\r\n` 区分，我们可以看到 在http  包头里 有 `content-length` 字段，``content-length` 用来确定 包体的长度，因此就可以划分哪些数据应该是一个请求中的，达到区分的目的。



参考HTTP 协议，一般 TCP 层 避免粘包，区分数据包主要有两种方式

1. 固定包头
2. 固定分隔符



### 固定包头

一般通用方式，固定包头 及 数据包的前几个字节 包含 数据包长度等信息，在处理中可以先读前几个字节，获取长度和其它扩展信息等，来确定后面包体的长度

#### 固定分隔符

即类似 HTTP 区分包头和包体的方式 用类似 \r\n 的方式来区分数据包