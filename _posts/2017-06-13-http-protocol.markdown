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

* 客户端请求报文中携带一个`Connection: Keep-Alive`首部
* 服务端的响应报文中携带一个`Connection: Keep-Alive`首部
