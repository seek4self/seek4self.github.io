---
title: tcp.md
top: false
cover: false
toc: true
mathjax: false
date: 2021-11-18 23:25:51
password:
summary:
categories:
tags:
---

## 三次握手

因为网络传输不可靠，tcp 握手是 c/s 双方为了验证对方收发消息的能力都正常

第一次：c->s 发送建立连接请求 SYN，若 s 能收到，则说明 c 可以正常发送消息，s 可以正常接收消息  
第二次: s->c 发送回应并建立从服务端到客户端的连接， 若 c 收到，则说明 s->c 可以正常发送回应消息，并且 c 可以正常收发消息  
第三次: c->s 发送回应消息，若 s 收到，说明，s 能正常 c 的消息  
从而确定cs双方都能收发消息
