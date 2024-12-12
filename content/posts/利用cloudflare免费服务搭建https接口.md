---
title: "利用cloudflare免费服务搭建https接口" 
date: 2024-12-12T13:02:38+08:00
draft: false
tags:
  - Aura
ShowToc: true
TocOpen: false 
---

# 问题描述

最近使用 [Vercel](https://vercel.com) 托管了一个 React 应用，其中使用 AJAX 请求从自建的 API 接口获取数据。但是前端是通过 HTTPS 连接的，这就要求请求 API 接口时也必须使用 HTTPS。这就涉及到了将 HTTP 升级为 HTTPS，并且还可能由于国内服务器默认封禁 80 | 443 | 8080 端口（需要备案），导致后端接口无法访问的问题（如果后端服务在国外则没有影响）。这篇文章记录了我利用 cloudflare 的自签名证书升级 HTTPS，Origin Rules 实现自定义域回源指定端口，以及利用 Nginx 的端口转发功能解决上述问题的过程。

本文默认读者对 cloudflare，Nginx 和云服务器的基本使用方法有所了解，因此不会再赘述其中的一些细节。



# 网络全链路流程图

![网络全链路流程图](https://s2.loli.net/2024/12/12/3uTWhPXa41nIQyp.png)



# 解决流程

## 1. cloudflare 自签发证书

使用 cloudflare 托管域名后，添加一条 A 类型记录指向部署后端服务的服务器的公网 IP，并开启 cloudflare 代理（小黄云）。在 cloudflare 的 SSL/TLS 管理中设置 SSL/TLS 加密模式为「完全严格」，在此模式下，浏览器到 cloudflare 和 cloudflare 到后端服务器之间全部采用 HTTPS。后者之间的 HTTPS 连接采用 cloudflare 自签发证书下发公钥。也就是说，cloudflare 在其中充当了可信的转发者角色。

接下来依次点击「源服务器 => 创建证书 => 创建」，就会得到证书（pem）和私钥（key），将这两段文本保存到后端服务器上的两个文件中，例如 /root/cert/example.com.pem 和 /root/cert/example.com.key。

## 2. 配置 cloudflare 自定义域回源指定端口

cloudflare 自定义域回源指定端口功能可以实现通过域名代理 「IP + 端口」，这在国内云服务器普遍封禁 443 端口时具有奇效。其实现原理就是通过 cloudflare 作为 HTTPS 连接的中间人，转发客户端浏览器和服务器之间通信，浏览器和 cloudflare 建立 HTTPS 连接，向自定义域名发送请求时，cloudflare 会解析域名的 IP，并将默认发送到服务器 443 端口的消息回源发送到服务器指定端口，从而绕开国内对 443 端口的封禁。

配置起来也很简单，在 cloudflare 中依次点击 「规则 => Origin Rules => 创建规则」，编辑表达式如下：

```
(http.host eq "此处填第一步 A 记录对应的域名，例如 example.com")

## 填写的完整表达式应该类似下面这个：
## (http.host eq "example.com")
```

接着配置 443 端口回源到哪个指定的端口，我这里指定的端口是 8082。



## 3. 在 Nginx 配置证书和端口转发

下载 Nginx 后，可以通过 `nginx -V` 命令查看默认配置文件路径（--config-path），我的路径是 /etc/nginx/nginx.conf，配置文件关键内容如下：

```
http {

	# 添加一个 server 块进行配置
    server {
		# 监听 HTTPS 连接端口，默认填 443，但是国内不备案会封禁
		# 这里填写的端口应和第二步最后指定的端口保持一致
        listen                  8082 ssl;
        # cloudflare 自签名证书签发的域名
        server_name             *.example.com;
        # 证书的绝对路径
        ssl_certificate         /root/cert/stop2think.top.pem;
        # 私钥的绝对路径
        ssl_certificate_key     /root/cert/stop2think.top.key;
        ssl_session_cache       shared:SSL:1m;
        ssl_session_timeout     5m;
        # 配置端口转发，ip 是后端服务部署机的 IP，port 是后端服务的端口
        location / {
                proxy_pass      http://ip:port;
        }
	}
}
```

`wq` 之后，使用命令 `nginx -t` 测试配置文件是否有误，配置检查通过后使用 `nginx -s reload` 使配置生效。至此，HTTPS 接口改造完成，问题解决！



# 我的项目

我的这个项目叫做 [Aura](https://aura.stop2think.top/)，是我根据自己的需求做（定制开发）的一个笔记回忆录，目前前端静态资源是托管在 [Vercel](https://vercel.com) 上的，感兴趣的可以看看。打开 F12 后查看网络可以看到 AJAX 请求后端使用的是 HTTPS。

