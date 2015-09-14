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

终于搬砖将自己的 Tumblr 重新搭好，然后发现中国移动把 Tumblr 墙了👀。只好用「反向代理」这种简单粗暴直接的方式实现无阻碍访问。

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
  
在我的 VPS 服务器上照此设置了 Nginx，然并卵，实际使用中所有的图片仍然显示不出来。  

---

### 分析

Tumblr 把图片分为两种：装饰网页用的底图、logo 等等，以及发表内容时上传的照片、动图等。前者被 Tumblr 统一放在了 `static.tumblr.com` 服务器上，后者所在的服务器则使用了 `数字.media.tumblr.com` 这种形式的子域名。  

事实上除了图片，Tumblr 还会在类似 `assets.tumblr.com` 这样的服务器上放置一些 JS 代码。所以，需要针对所有这些服务器设置反向代理。

---

### 改进

利用 Nginx 的 `sub_filter`，先把页面中所有的 `tumblr.com` 替换为 `static.xXx.com`，即添加这行：
<pre><code>sub_filter tumblr.com xXx.com;</code></pre>
(注#1：并不是所有 Nginx 的版本都支持超过一个 `sub_filter`，所以请更新 Nginx 到最新版。)  
(注#2：个人域名的 DNS 服务商最好支持 `catch-all`，即 `*.xXx.com` 这种形式的解析，这样就不用额外处理每一个子域名。)  

然后在配置尾部添加这一段：  

    server
    {
    listen 80;
    server_name static.xXx.com;
    
    location / {
    proxy_pass http://static.tumblr.com;
    proxy_redirect off;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Accept-Encoding "";
    }
    
    }

重启 Nginx 并测试，logo 和 底图都出来了，说明思路正确。

下一步处理 `数字.media.tumblr.com`。这里麻烦的地方是里面的 `数字` 并不确定具体是多少，按上面的方法将所有的数字组合都设置一遍会显得比较蠢。换一种思路，其实并不需要知道具体数字是多少，只需替换确定的部分，即 `media.tumblr.com` 即可。添加一行：
    sub_filter 'media.tumblr.com' 'media.xXx.com';

以及尾部这一段：

    server
    {
    listen 80;
    server_name ~^(?<subdomain>\w+)\.media\.xXx\.com$;
    
    location / {
    resolver 8.8.8.8;
    proxy_pass http://$subdomain.media.tumblr.com;
    proxy_redirect off;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Accept-Encoding "";
    }
    
    }

解释一下：由于并不知道具体的 `数字` 是多少，这里使用了正则表达式和变量来替换 `数字` 部分。另外，这里必须加一行 `resolver` 来作 DNS 解析否则 Nginx 会报错。

按照这个思路，再添上

---

### 进阶

之前提到，上面的方案有一个「不完美」的地方 － 现在其实同时存在着两个内容完全相同的网站: 一个是 `xXx.tumblr.com`，一个是 `blog.xXx.com`，虽然后者只是前者的反向代理，但对搜索引擎来说这是两个网站。

---

### 安装RubyGems

官网下载 [http://rubygems.org/pages/download](http://rubygems.org/pages/download) rubygems-2.4.5.zip   

cd到RubyGems目录   

![ruby-gems]({{ "/css/pics/ruby-gems.png"}})    

执行安装   

![ruby-gems-setup]({{"/css/pics/ruby-gems-setup.png"}})   

---

### 用RubyGems安装Jekyll

执行下面的语句安装   

![jekyll-setup]({{"/css/pics/jekyll-setup.png"}})   

安装结束画面   

![jekyll-setup-finish]({{"/css/pics/jekyll-setup-finish.png"}})   

至此jekyll就已经安装完毕了，后续就是个性化的自己设定了。   

---

### 创建博客

在d盘新建一个工作区jekyllWorkspace

cd到jekyllWorkspace   

执行jekyll new name创建新的工作区   

![jekyllWorkSpace]({{"/css/pics/jekyllWorkSpace.png"}})   

文件结构如下：   

![jekyllFiles]({{"/css/pics/jekyllFiles.png"}})

cd到博客文件夹，开启服务器   

![serve]({{"/css/pics/serve.png"}})   

watch为了检测文件夹内的变化，即修改后不需要重新启动jekyll

我的环境下启动报错(你的可能没有)，再安装yajl-ruby和rouge  

![yajl]({{"/css/pics/yajl.png"}})

再次启动服务器成功

![serve-sucess]({{"/css/pics/serve-sucess.png"}})

访问 http://localhost:4000/   

![browser]({{"/css/pics/browser.png"}})   

详细文章页面   

![browser2]({{"/css/pics/browser2.png"}})  

---

##后续 

*  整个安装过程参考了jekyll官网，注意jekyll还有一个简体中文官网，不过比较坑（我就被坑了），有些内容没有翻译过来，有可能会走弯路，建议如果想看中文的相关资料，也要中英对照着阅读。[jekyll中文网 http://jekyllcn.com](http://jekyllcn.com), [jekyll英文网 http://jekyllrb.com](http://jekyllrb.com)
*  jekyll中的css是用sass写的，当然直接在`_sass/_layout.scss`中添加css也是可以的。
*  本文是用Markdown格式来写的，相关语法可参考： [Markdown 语法说明 (简体中文版) http://wowubuntu.com/markdown/](http://wowubuntu.com/markdown/)  
*  按照本文的说明搭建完博客后，用`github Pages`托管就可以看到了。注意，在github上面好像不支持rouge，所以要push到github上时，我将配置文件_config.yml中的代码高亮改变为`highlighter: pygments`就可以了
*  博客默认是没有评论系统的，本文的评论系统使用了多说，详细安装办法可访问[多说官网 http://duoshuo.com/](http://duoshuo.com/)，当然也可以使用[搜狐畅言 http://changyan.sohu.com/](http://changyan.sohu.com/)作为评论系统。	
*  也可以绑定自己的域名，如果没有域名，可以在[godaddy http://www.godaddy.com/](http://www.godaddy.com/)上将域名放入购物车等待降价，买之。
*  祝各位新年快乐！

---

## 可能出现的问题

### `hitimes/hitimes (LoadError)`

**错误代码：**

<pre><code class="markdown">C:/Ruby22/lib/ruby/2.2.0/rubygems/core_ext/kernel_require.rb:54:in `require': cannot load such file -- hitimes/hitimes (LoadError)</code></pre>

**解决方法：**

在stackoverflow上又一个很好的解决方法。[hitimes require error when running jekyll serve on windows 8.1](http://stackoverflow.com/questions/28985481/hitimes-require-error-when-running-jekyll-serve-on-windows-8-1) 虽然上面的题主问的是 win 8.1 系统下的情况，但是同样适用于 win7。下面我简单翻译一下错误原因和解决方法。

> 可能是 Ruby 2.2 和 hitimes-1.2.2-x86-mingw32 中有一些 ABI 变化，少了一些相关的类库。
> 
> 所以卸载 hitimes 并通过 `--platform ruby` 重装即可。代码如下：
>
><pre><code class="markdown">gem uni hitimes
**Remove ALL versions**
gem ins hitimes -v 1.2.1 --platform ruby
</code></pre>
> 然后将自动重新编译 hitimes 并适用于 Ruby 2.2

下面是我自己的卸载和安装过程：

<pre><code class="markdown">E:\GitWorkSpace\gaohaoyang.github.io>gem uni hitimes

You have requested to uninstall the gem:
        hitimes-1.2.2-x86-mingw32

timers-4.0.1 depends on hitimes (>= 0)
If you remove this gem, these dependencies will not be met.
Continue with Uninstall? [yN]  y
Successfully uninstalled hitimes-1.2.2-x86-mingw32

E:\GitWorkSpace\gaohaoyang.github.io>gem ins hitimes -v 1.2.1 --platform ruby
Fetching: hitimes-1.2.1.gem (100%)
Temporarily enhancing PATH to include DevKit...
Building native extensions.  This could take a while...
Successfully installed hitimes-1.2.1
Parsing documentation for hitimes-1.2.1
Installing ri documentation for hitimes-1.2.1
Done installing documentation for hitimes after 1 seconds
1 gem installed</code></pre>


关于，[hitimes](https://rubygems.org/gems/hitimes/versions/1.2.2) 是一个快速的高效的定时器解决方案库，详情可以去官网查看。


