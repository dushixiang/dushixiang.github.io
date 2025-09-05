+++
date = '2025-09-05T17:03:53+08:00'
draft = false
title = 'Nginx 二级目录单页应用的正确配置方式'
+++

今天配置了一个 Nginx 的二级目录单页应用，搞了好久才配置好，记录一下。

正确的配置方式示例：

```shell
# 子应用
location /subapp/ {
    alias /opt/app/subapp/; 
    index index.html;
    try_files $uri /subapp/index.html;
}

# 根应用
location / {
    root   /opt/app/web;
    index  index.html index.htm;
    try_files $uri /index.html;
}
```



### 坑1: 使用 `alias` 而不是 `root`

Nginx 的 root 会把 location 的匹配部分也拼接进去，所以访问：

```shell
https://yourdomain/subapp/
```
实际会去找：

```shell
/opt/app/subapp/subapp/index.html
```

这就多了一层 `subapp` 目录，通常会 404。

注意 alias 最后必须加 `/` 。

### 坑2: try_files 地址要写 `/subapp/index.html`

之前写的 `/index.html` 让我误以为是被 `/` 根应用拦截了，实际上是找错 html 了。