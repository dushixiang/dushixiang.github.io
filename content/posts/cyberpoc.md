+++
date = '2025-09-06T00:00:00+08:00'
draft = false
title = '我为什么开源了三年前写的漏洞验证平台？'
+++

大概三年前，我看到有人用代码片段分享漏洞原理，觉得很有意思，我也模仿设计了一些 Golang 的漏洞。

可惜想要验证，得自己搭建环境，门槛不低。

于是我又做了一个平台，把漏洞环境封装成 Docker 镜像。用户只需点一下，就能在几秒内启动环境，直接验证。

第一个版本很快上线，我陆续封装了十几个漏洞环境。但后来越做越复杂，加了虚拟机管理和分布式网络，结果部署维护麻烦到连我自己都不想搞了，再加上我了解到的 Golang 漏洞就那么多，平台没人用，项目也就搁置了。

最近我做了一个违背祖宗的决定：去掉多余的东西，只保留容器化的漏洞环境，并把平台和漏洞环境全部开源。

项目和漏洞环境都是用 Golang 写的，配置要求很低，开箱即用。
如果你觉得有用，欢迎点个 star：

- 漏洞环境：https://github.com/dushixiang/vulnerable-code
- 验证平台：https://github.com/dushixiang/cyberpoc
- 测试地址：https://cyberpoc.typesafe.cn

附上几张截图

![](/images/cyberpoc_vuln.png)
![](/images/cyberpoc_0.png)
![](/images/cyberpoc_1.png)