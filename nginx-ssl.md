title: Nginx SSL配置  
date: 2014/8/30 20:00:00
tags: [nginx]
categories: [middleware]
---

## 安装nginx ssl模块  

如果配置ssl启动nginx的时候提示类似下面这种错误, 那应该是使用的nginx没有安装ssl模块.  
> nginx: [emerg] the "ssl" parameter requires ngx_http_ssl_module in /usr/local/nginx/conf/server.conf:6 

nginx默认不会安装ssl模块, 如果没有安装需要修改编译参数重新编译安装.  
``` sh  
./configure --with-http_stub_status_module --with-http_ssl_module  
make  
make install  
```
<!--more-->

## 配置nginx ssl  

主要是参考[这篇文章](http://my.oschina.net/zhlmmc/blog/42125)配置.  
文章也比较详细, 但是部署证书那里作者没有说清楚, 这一部分也是比较坑的, 我把碰到的问题记录下.  
(我这里配的是**自签名证书**, 相对于认证的证书少一个步骤, 整体都差不多)  

另外文章介绍的是分别在nginx和tomcat上都配置https, 实际上可以只在nginx上配置https, 然后转发给tomcat的http端口, 这样效率会更高一些.  

自签名证书主要是用的java自带的[Keytool](http://www.cnblogs.com/diyunpeng/archive/2012/02/20/2360226.html)  

### 生成keystore  

``` sh
keytool -genkey -keystore keystore.jks  
```
按照命令行提示依次输入即可, 密码要记住.  

### 生成证书文件crt

``` sh
keytool -list -rfc -keystore keystore.jks 
```
输入密码以后即可看到类似下面的内容:  
> -----BEGIN CERTIFICATE-----  
> 一堆乱码...  
> -----END CERTIFICATE-----  

这一串东西就是证书了, 拷贝下来另存为\*.crt文件即可.   
**注意:**  "BEGIN..."和"END..."那两行也是要的.  

### 导出私钥key  

文章里有说明, 使用[java-exportpriv](https://code.google.com/p/java-exportpriv/)这个工具导出.  
导出以后可以看到类似下面的内容, 把私钥部分另存为文件即可.  
> -----BEGIN PRIVATE KEY-----  
> 一堆乱码...  
> -----END PRIVATE KEY-----  

**注意:** "BEGIN..."和"END..."那两行也是要的.  

### 关于keystore的alias  

导出私钥的时候需要指定keystore的alias, 如果生成keystore的时候没有指定默认就是mykey.  
如果不小心手残随便填了一个, 或者是脑残忘了, 或者是像我一样是别人设置的没有告诉我, 不用怕, 用上面的-list命令可以也可以找到.  
如果是密码也这么搞忘了, 那就哭吧... :-)


### 配置ssl  
主要是证书和私钥这里比较坑, 搞定以后配置就很简单了.  
``` sh
server {
        listen       443 ssl;
        ssl on;
        ssl_certificate     /usr/local/nginx/conf/ssl.crt;
        ssl_certificate_key /usr/local/nginx/conf/ssl.key;
 }
```

### 相关异常  

- 提示类似这样错误说明证书配置有问题:  
> [emerg] 21482#0: PEM_read_bio_X509_AUX("/usr/local/nginx/conf/ssl.crt") failed (SSL: error:0906D06C:PEM routines:PEM_read_bio:no start line:Expecting: TRUSTED CERTIFICATE)


- 提示类似这样的错误说明key有问题:  
> [emerg] SSL_CTX_use_PrivateKey_file("/usr/local/nginx/conf/ssl.key") failed (SSL: error:0906D06C:PEM routines:PEM_read_bio:no start line:Expecting: ANY PRIVATE KEY error:140B0009:SSL routines:SSL_CTX_use_PrivateKey_file:PEM lib)