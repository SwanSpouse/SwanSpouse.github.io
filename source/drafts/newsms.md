---
title: 短信网关迁移
date: 2018-09-12 23:46:41
tags:
  - business logic
---

### TODO list

首先说清任务需求和难点。

然后解决办法。

遇到的问题。

整理url这一些列，还是有urlencode decode 加密解密 这些业务逻辑

这个等我解决完一切问题再说吧。

或者可以简单把lain说一下。自动化部署 构建这一套流程。CI流程。

### URL地址解释

![url](https://s1.ax1x.com/2018/09/14/iEkq5d.png)

scheme,protocol: 指定因特网服务的类型。例如:HTTP,FTP
host:   指定此域名中的主机。HTTP默认主机是www
port:   指定主机的端口号。HTTP默认端口号是80
path:   指定远端服务器上的路径。默认被指定到网站的根目录。
file:   指定远端服务器上的文件。
extension: 远端服务器上的文件格式。例如.html
query:  请求所携带的参数

#### golang 中解析url的各种方法
