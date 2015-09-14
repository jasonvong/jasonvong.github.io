---
layout: post
title:  "反向代理 Tumblr"
date:   2015-09-13 12:02
categories: Nginx Tumblr
excerpt: 利用 DNSPod 的多线路解析，通过 Nginx 实现对 Tumblr 的反向代理。
---

* content
{:toc}


## 序

终于搬砖将自己的 Tumblr 重新搭好，然后发现中国移动把 Tumblr 墙了👀。决定用「反向代理」这种简单粗暴直接的方式实现无阻碍访问。

---

## 过程

### 初始设置

一般都会用 Nginx 来做反向代理，这个我不熟，所以必须先搜索现成的配置来用，再根据实际情况边学边改。找到了一个反向代理 Tumblr 的配置：
<pre><code>server
{
listen 80;
server_name blog.xXx.com;      

location / {
proxy_pass http://xXx.tumblr.com;
proxy_redirect off;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header Accept-Encoding "";
}

sub_filter xXX.tumblr.com blog.xXx.com;
sub_filter_once off;
}</code></pre>
(via [Wood Tale](http://adaromu.tumblr.com/post/33722081482/nginx反向代理tumblr配置))  
  
在自己的 VPS 服务器上照此设置了 Nginx，然并卵，实际使用中所有的图片仍然显示不出来。  

---

### 问题分析

Tumblr 把图片分为两种：装饰网页用的底图、logo 等等，以及发表内容时上传的照片、动图等。前者被 Tumblr 统一放在了 `static.tumblr.com` 服务器上，后者所在的服务器则使用了 `数字.media.tumblr.com` 这种形式的子域名。  

事实上除了图片，Tumblr 还会在类似 `assets.tumblr.com` 这样的服务器上放置一些 JS 代码。所以，需要针对所有这些服务器设置反向代理。

---

### 改进设置

利用 Nginx 的 `sub_filter`，先把页面中所有的 `tumblr.com` 替换为 `static.xXx.com`，即添加这行：
<pre><code>sub_filter tumblr.com xXx.com;</code></pre>
(注#1：并不是所有 Nginx 的版本都支持超过一个 `sub_filter`，所以请更新 Nginx 到最新版。)  
(注#2：个人域名的 DNS 服务商最好支持 `catch-all`，即对 `*.xXx.com` 这种形式的解析，这样就不用额外处理每一个子域名。)  

之后利用正则表达式和变量，配置对上述服务器的反向代理：

    server
    {
    listen 80;
    server_name ~^(?<subdomain>\S+)\.xXx\.com$;
    
    location / {
    resolver 8.8.8.8;
    proxy_pass http://$subdomain.tumblr.com;
    proxy_redirect off;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Accept-Encoding "";
    }
    
    }

(这里必须加一行 `resolver` 来作 DNS 解析，否则 Nginx 会报错。)

经测试，这样配置后图片就可以显示出来了。

---

### 进阶

上面的方案有一个「不完美」的地方，即 `xXx.tumblr.com` 仍可被单独访问（特别是在墙外）。虽然 `blog.xXx.com` 只是反向代理，但对搜索引擎来说它们是内容完全相同但域名不同的两个网站，不利于 SEO。既然 Tumblr 支持个人域名，所以最好的办法就是实现「一个域名同时指向 Tumblr 和 Nginx 所在的反向代理服务器」。DNSPod 就提供这种多线路解析的服务，即国外访问 `blog.xXx.com` 时，返回的是 Tumblr 的服务器 IP，而国内访问 `blog.xXX.com` 时，返回的是反向代理服务器 IP。

综上，进一步修改 Nginx 配置的完整结果：

    server
    {
    listen 80;
    server_name blog.xXx.com;
    
    location / {
    proxy_pass http://66.6.44.4;
    proxy_redirect off;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header Accept-Encoding "";
    }
    
    sub_filter 'tumblr.com' 'xXx.com';
    sub_filter_once off;
    }
    
    
    server
    {
    listen 80;
    server_name ~^(?<subdomain>\S+)\.xXX\.com;
    
    location / {
    resolver 8.8.8.8;
    proxy_pass http://$subdomain.tumblr.com;
    proxy_redirect off;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Accept-Encoding "";
    }
    
    }


---

## 总结 

*  个人域名 `xXx.com`，并将域名 DNS 服务商托管给 DNSPod；
*  在 Tumblr 上将 URL 设为个人域名：`blog.xXx.com`；
*  VPS 或任何可安装 Nginx 的服务器，按上述进行设置；
*  在 DNSPod 上将 `blog.xXx.com` 设为多线路解析，国内解析到 Nginx 服务器 IP，默认则解析到 Tumblr 的 IP；
*  同时在 DNSPod 上将 `*.xXx.com` 解析到 Nginx 服务器 IP。

---

## 参考
http://adaromu.tumblr.com/post/33722081482/nginx反向代理tumblr配置
http://jyorr.com/post/4085366506/reverse-proxying-for-tumblr-w-nginx
http://www.storyday.com/html/y2012/3165_on-tumblr-reverse-agent.html
http://webmasters.stackexchange.com/questions/55698/nginx-reverse-proxy-for-tumblr


