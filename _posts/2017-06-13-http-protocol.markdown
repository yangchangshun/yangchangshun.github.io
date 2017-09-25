---
layout:     keynote
title:      "HTTP协议"
subtitle:   "HTTP长连接"
navcolor:   "invert"
date:       2017-06-13
author:     "Sean"
tags:
    - HTTP
---


> HTTP长连接主要为了解决http性能问题


`持久化连接`
* 客户端请求报文中携带一个`Connection: Keep-Alive`首部
* 服务端的响应报文中携带一个`Connection: Keep-Alive`首部

`管道化连接`
> HTTP/1.1支持在持久连接上发送管道请求，在响应发送到对端之前，就可以接着发送后面的请求了。
> 注意：不要把非幂等请求在管道中发送（如：POST请求)


长连接的相关问题
* 盲中继
* Netscape发明了一种`Proxy-Connection`只能解决一级明确的盲代理问题
