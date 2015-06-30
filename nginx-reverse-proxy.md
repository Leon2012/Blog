title: Nginx搭建反向代理服务
date: 2015/6/28 20:00:00
tags: [nginx,proxy]
categories: [middleware]
---

公司办公电脑因为网络安全管控的原因, 很多网站都被屏蔽了.
好在自己的域名没有被封, 所以用Nginx自建反向代理, 访问google, github等网站.

下面会记录一下Nginx搭建反向代理以及折腾nginx配置过程中踩过的坑.

## Nginx编译安装
nginx的安装教程网上也有很多, 这里就不赘述了.  
主要关注下nginx的模块, 使用`nginx -v`命令可以看到当前nginx支持的模块:
> configure arguments: --with-pcre=../pcre-8.37/ --with-http_sub_module --with-http_stub_status_module --with-http_ssl_module

- --with-pcre: urlrewrite模块依赖pcre
- --with-http_sub_module: 反向代理替换链接的时候会用到
- --with-http_stub_status_module: Nginx状态监控
- --with-http_ssl_module: 支持https ssl

<!--more-->

## Nginx反向代理配置
主要用到[http_proxy](http://nginx.org/en/docs/http/ngx_http_proxy_module.html)模块.
[官方wiki也有一些例子](http://wiki.nginx.org/Configuration#Proxying_examples)可以参考.  
[中文也有例子](https://www.centos.bz/2014/06/nginx-proxy-google/), 后面我也会贴出我自己配置好的可以参考.

### http_sub_module
这个模块主要用于替换字符, 例如github引用了assets-cdn域名下的静态资源文件, 那我就要把assets-cdn替换到我自己的域名, 再对这个域名做反向代理.

> The ngx_http_sub_module module is a filter that modifies a response by replacing **one** specified string by another.  

参考资料:
- [官方文档](http://nginx.org/en/docs/http/ngx_http_sub_module.html)
- [Nginx替换网站响应内容](http://www.ttlsa.com/linux/nginx-modules-ngx_http_sub_module/)

sub_module只能支持替换一类字符, 如果有替换多类字符的场景可以用[HttpSubsModule](http://wiki.nginx.org/HttpSubsModule)

> nginx_substitutions_filter is a filter module which can do both regular expression and fixed string substitutions on response bodies. This module is quite different from the Nginx's native Substitution Module. It scans the output chains buffer and matches string line by line, just like Apache's mod_substitute.

如果碰到字符替换不生效的情况, 首先确认下response中是否包含你要替换的字符.

我就踩过这个坑, 是因为反向代理抓回的内容被压缩了找不到对应的字符.

**解决办法:**  
配置反向代理的时候让对方服务器返回原始未压缩的报文:
```
proxy_set_header       Accept-Encoding "";
```

### 缓存配置
缓存可以减少重复请求对方服务器, 提高访问速度.  
配置参考[Reverse Proxy with Caching](http://wiki.nginx.org/ReverseProxyCachingExample)

### 授权验证
如果自己搭建的反向代理不想让别人发现或者不想被搜索引擎找到, 可以做一些权限验证.  
有个简单的办法: 通过一个静态页面中的JS脚本写授权token到cookie, 然后在nginx配置中验证.
- js读url参数中的token, 写到cookie
```js
function setCookie(){
    var url = location.search;
    if (url.indexOf("?") != -1) {
        var str = url.substr(1);
        strs = str.split("&");
        for(var i = 0; i < strs.length; i ++) {
            var k = strs[i].split("=")[0];
            var v = unescape(strs[i].split("=")[1]);
            var date = new Date();
            date.setTime(date.getTime() + 30 * 24 * 3600 * 1000);
            document.cookie = k + "=" + v + "; domain=.pingan.im; path=/; expires = " + date.toGMTString();
        }
    }
}
```
- nginx读不到cookie返回401
```java
if ($http_cookie !~* "token=xxxxx"){
    return 401;
}
```

### nginx中的if嵌套逻辑判断
nginx中的if功能相对弱一些, 不支持and/or条件判断, 也不支持嵌套, 但是可以用变量的方式来变通实现.

例如通常我们这么写if
``` nginx
if (a && !b){
    return 401;
}
```
那nginx里就要变成这样了
``` nginx
set $flag = false;
if(a){
    $flag = true;
}
if(!b){
    $flag = true;
}
if($flag = "true"){
    return 401;
}
```
所以如果条件多一些那相应的代码也会更复杂一些.

### nginx内置变量
nginx有很多内置的变量, 在配置的时候经常会用到.  
[官方文档有详细列表和介绍](http://nginx.org/en/docs/http/ngx_http_core_module.html#variables)


## 其他
以下一些配置跟反向代理没有很大关系, 也顺便记录一下.

### http_stub_status_module
> The ngx_http_stub_status_module module provides access to basic status information.

监控nginx状态很方便.

参考资料:
- [官方文档](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html)
- [Nginx 开启 stub_status 模块监控](http://blog.csdn.net/yangjiehuan/article/details/7030290)

### gzip压缩
依赖[gzip_module](http://nginx.org/en/docs/http/ngx_http_gzip_module.html), nginx默认会安装, 开启gzip可以节省不少流量, 提升访问速度.

参考配置:
``` nginx
gzip on;
gzip_min_length 1k;
gzip_comp_level 6;
gzip_types text/plain application/javascript text/css application/xml;
```

### 配置https证书
配置https支持ssl加密传输, 对我们普通人来说*然并卵*, 但是至少看起来高大上啊 :-)

- 证书申请

有收费的也有免费的, 免费的可以用[startssl](https://www.startssl.com/), 可以参考网上的[介绍](http://www.freehao123.com/startssl-ssl/)去申请.
虽然申请过程有点繁琐, 但是坚持一下等你折腾完看到浏览器导航栏上那个绿色小锁的时候还是小有成就感的 :-)  
![https://pingan.im](http://7ktufg.com1.z0.glb.clouddn.com/2015062801.jpg)

- 配置

证书申请下来, 配置就很简单了, 监听443端口, 转发到80端口上去处理就行了, 省得再配置一次.
``` nginx
server {
    listen              443 ssl;
    ssl                 on;
    ssl_certificate     /***/nginx/conf/ssl.crt;
    ssl_certificate_key /***/nginx/conf/ssl.key;

    location / {
        proxy_pass      http://localhost;
    }

}
```

## 配置参考
最后贴一下我目前的反向代理配置

- google

``` nginx
server {
    listen                     80;
    server_name                g.pingan.im;

    access_log                 logs/g.access.log;
    error_log                  logs/g.error.log;

    include   /***/nginx/conf/props.conf;

    location / {
        include                 /***/nginx/conf/auth.conf;

        proxy_pass             https://www.google.co.jp;
        proxy_cache            STATIC;
        proxy_cache_valid      200 302  1h;
        proxy_cache_valid      404      1m;
        proxy_cache_use_stale  error timeout invalid_header updating http_500 http_502 http_503 http_504;
        proxy_redirect         https://www.google.co.jp/ /;
        proxy_set_header       Accept-Encoding "";
        proxy_set_header       Accept-Language "zh-CN";
        proxy_set_header       User-Agent $http_user_agent;
        sub_filter             www.google.co.jp g.pingan.im;
        sub_filter_once        off;
    }

    include   /***/nginx/conf/error_page.conf;
}
```

- github, gist

```
server {
    listen        80;
    server_name   git.pingan.im;

    access_log    logs/git.access.log;
    error_log     logs/git.error.log;

    include   /***/nginx/conf/props.conf;

    location / {
        include                 /***/nginx/conf/auth.conf;

        proxy_pass              https://github.com;
        proxy_cache             STATIC;
        proxy_cache_valid       200 302  1h;
        proxy_cache_valid       404      1m;
        proxy_cache_use_stale   error timeout invalid_header updating http_500 http_502 http_503 http_504;
        proxy_redirect          https://www.github.com/ /;
        proxy_set_header        Accept-Encoding "";
        proxy_set_header        Cookie $git_cookie;
        proxy_set_header        Accept-Language "zh-CN";
        proxy_set_header        User-Agent $http_user_agent;       
        sub_filter              https://assets-cdn.github.com http://git.pingan.im;
        sub_filter_once         off;
    }

    include   /***/nginx/conf/error_page.conf;
}
server {
    listen        80;
    server_name   gist.pingan.im;

    access_log    logs/git.access.log;
    error_log     logs/git.error.log;

    include   /***/nginx/conf/props.conf;

    location / {
        include                 /***/nginx/conf/auth.conf;

        proxy_pass              https://gist.github.com;
        proxy_cache             STATIC;
        proxy_cache_valid       200 302  1h;
        proxy_cache_valid       404      1m;
        proxy_cache_use_stale   error timeout invalid_header updating http_500 http_502 http_503 http_504;
        proxy_redirect          https://gist.github.com/ /;
        proxy_set_header        Accept-Encoding "";
        proxy_set_header        Cookie $gist_cookie;
        proxy_set_header        Accept-Language "zh-CN";
        proxy_set_header        User-Agent $http_user_agent;
        sub_filter              https://gist-assets.github.com http://gist.pingan.im;
        sub_filter_once         off;
    }

    include   /***/nginx/conf/error_page.conf;
}
```

- 以上配置都include的auth.conf配置文件, 用于校验权限

``` nginx
set $auth_flag true;
if ($http_cookie !~* "token=paic3234"){
    set $auth_flag false;
}
# 因为反向代理github的时候引用的资源文件用到了"crossorigin"这个属性, 不会把cookie带上来, 所以这里要特殊处理
if ($uri ~ "/assets/"){
    set $auth_flag true;
}
if ($auth_flag = "false"){
    return 401;
}
```