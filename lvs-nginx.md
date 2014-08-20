title: LVS & Nginx
date: 2014/8/10 20:00:00
tags: [lvs,nginx]
categories: [middleware]
---

使用LVS + Keepalived + Nginx + Tomcat的组合搭建高性能高可用web服务, 分别发挥各自优势:  

- LVS  
  - 工作在网络四层, 抗负载能力很强.  
  - 有DR, NAT, TUN三种模式:  
    DR模式是最常用的, 也是性能最好的;  
    DR模式不支持端口映射, NAT模式可以支持但是性能相对差一些, 可以用nginx代替;  
    TUN模式一般很少用;  
- Keepalived  
  - 通常都是和lvs一起使用的, 支持lvs双机热备; 也可以独立出来搭配nginx等其他负载均衡软件使用.  
  - 支持real server的故障隔离以及负载均衡机器之间的失败切换.  
- Nginx  
  - 工作在网络七层应用层, 抗负载能力相对LVS弱一些, 但是配置更灵活, 支持端口映射, 支持复杂的正则表达式处理.  
  - 可以支持动静分离, 本身可以做静态服务器.  
  - 可以根据二级域名和子目录进行分流, 可以在一个域名下挂多个Web应用.  
- Tomcat  

参考资料:  
- [Keepalived官网推荐的lvs集群文档](http://www.keepalived.org/pdf/sery-lvs-cluster.pdf)  
- [LVS和Nginx比较分析](http://my.oschina.net/wmsjhappy/blog/273754)  
- [各类负载均衡软件的简介和对比](http://my.oschina.net/wyyft/blog/169120)  
- [基于LVS负载均衡的高性能Web站点设计与实现](http://my.oschina.net/alanlqc/blog/151395)  

以下简单记录一下LVS和Ngix的安装和配置, 以及遇到的问题.  

## LVS  

### 安装LVS 
安装ipvsadm和keepalived

参考资料:  
- [基于LVS负载均衡的高性能Web站点设计与实现](http://my.oschina.net/alanlqc/blog/151395)  
- [RHEL 5.4下部署LVS(DR)+keepalived实现高性能高可用负载均衡](http://www.cnblogs.com/mchina/archive/2012/05/23/2514728.html)


#### 安装ipvsadm的时候提示类似  
  > ip_vs.h:15:29: error: netlink/netlink.h: No such file or directory

  这样的错误, 是因为缺少依赖的软件包  

  解决办法: 安装相关依赖软件包即可.  

  参考资料:    
  - [解决CentOS 6.2下安装ipvsadm-1.26报错](http://www.linuxidc.com/Linux/2012-03/57386p2.htm)  
  - [LVS 单独完成--负载均衡](http://blog.sina.com.cn/s/blog_5f54f0be0101eyiu.html)


### 配置LVS  

参考资料:  
[ipvsadm命令参考](http://zh.linuxvirtualserver.org/node/5)  
[LVS三种模式](http://www.uml.org.cn/zjjs/201211124.asp)  
[LVS十种调度算法介绍](http://www.cnblogs.com/luxf/archive/2011/01/11/1932972.html)  

#### 启动lvs 

可以参考以下脚本启动:  
``` shell  
#!/bin/sh
VIP=192.168.1.100
RIP1=192.168.1.9
RIP2=192.168.1.10
. /etc/rc.d/init.d/functions
case "$1" in
start)
  echo "start LVS of DirectorServer"
  /sbin/ipvsadm --set 30 5 60
  #Set the Virtual IP Address
  /sbin/ifconfig eth0:0 $VIP broadcast $VIP netmask 255.255.255.255 up
  /sbin/route add -host $VIP dev eth0:0
  #Clear IPVS Table
  /sbin/ipvsadm -C
  #Set Lvs
  /sbin/ipvsadm -A -t $VIP:80 -s wlc
  /sbin/ipvsadm -a -t $VIP:80 -r $RIP1:80 -g
  /sbin/ipvsadm -a -t $VIP:80 -r $RIP2:80 -g
  touch /var/lock/subsys/ipvsadm > /dev/null 2>&1
  ;;
stop)
  echo "close LVS Directorserver"
  /sbin/ipvsadm -C
  /sbin/ipvsadm -Z
  /sbin/route del $VIP
  /sbin/ifconfig eth0:0 down
  rm -rf /var/lock/subsys/ipvsadm > /dev/null 2>&1
  ;;
status)
  if [ ! -e /var/lock/subsys/ipvsadm ];then
  echo "ipvsadm stopped!"
  exit 1
  else
  echo "ipvsadm started!"
  fi
  ;;
*)
  echo "Usage: $0 {start|status|stop}"
  exit 1
esac
exit 0
```  

启动以后就可以用ifconfig验证绑定的vip.  


#### keepalived配置  

**注意:** 网上有很多例子是通过脚本启的, 分别用脚本启动lvs和realserver, 但是如果使用keepalived就不要再用脚本启动lvs了, 否则会冲突.  

Keepalived配置文件可以参考以下模板修改, 注意Master和Backup机器之间的state和priority需要修改, 其他都保持一致.  

``` shell  
! Configuration File for keepalived
global_defs {
   router_id LVS_DEVEL
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.100
    }
}
virtual_server 192.168.1.100 80 {
    delay_loop 6
    lb_algo wlc
    lb_kind DR
    persistence_timeout 50
    protocol TCP

    real_server 192.168.1.9 80 {
        weight 3
        TCP_CHECK {
            connect_timeout 10
            nb_get_retry 3
            delay_before_retry 3
            connect_port 80
        }
    }
    real_server 192.168.1.10 80 {
        weight 3
        TCP_CHECK {
            connect_timeout 10
            nb_get_retry 3
            delay_before_retry 3
            connect_port 80
        }
    }
}
```  

Keepalived配置为服务, 启动以后用ip a就可以看到绑定的端口(**不用ifconfig**).  


#### 启动real server  

real server上无需安装lvs相关组件, 只要参考执行以下脚本即可:  
```shell  
#!/bin/bash
VIP=192.168.1.100
. /etc/rc.d/init.d/functions
case "$1" in
start)
  /sbin/ifconfig lo:0 $VIP netmask 255.255.255.255 broadcast $VIP
  /sbin/route add -host $VIP dev lo:0
  echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
  echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
  echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
  echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
  sysctl -p >/dev/null 2>&1
  echo "RealServer Start OK"
  ;;
stop)
  /sbin/ifconfig lo:0 down
  /sbin/route del $VIP >/dev/null 2>&1
  echo "0" >/proc/sys/net/ipv4/conf/lo/arp_ignore
  echo "0" >/proc/sys/net/ipv4/conf/lo/arp_announce
  echo "0" >/proc/sys/net/ipv4/conf/all/arp_ignore
  echo "0" >/proc/sys/net/ipv4/conf/all/arp_announce
  echo "RealServer Stoped"
  ;;
status)
  # Status of LVS-DR real server.
  islothere=`/sbin/ifconfig lo:0 | grep $VIP`
  isrothere=`netstat -rn | grep "lo:0" | grep $VIP`
  if [ ! "$islothere" -o ! "isrothere" ];then
  # Either the route or the lo:0 device not found.
  echo "LVS-DR real server Stopped."
  else
  echo "LVS-DR Running."
  fi
  ;;
*)
  echo "Usage: $0 {start|status|stop}"
  exit 1
esac 
exit 0
```

Real server启动以后就可以用vip访问进行验证了.  

## Nginx

### Nginx安装  

参考[nginx安装教程](http://www.nginx.cn/install)

- 安装pcre提示错误:  
  > configure: error: You need a C++ compiler for C++ support.  

  解决办法:  
  [yum install gcc](http://blog.sina.com.cn/s/blog_7253980a0101guiz.html)  

- 启动nginx提示错误:  
  > error while loading shared libraries: libpcre.so.1: cannot open shared object file: No such file or directory  

  解决办法: 
  [进入lib目录手动链接](http://www.cnblogs.com/wenanry/archive/2012/04/16/2451881.html)
  > ln -s libpcre.so.0.0.1 libpcre.so.1

### Nginx配置  

nginx配置比较简单, 反向代理可以参考以下配置:  
1. 在http节点下新增real server配置  
```shell  
upstream hello {
    #ip_hash;
    server 192.168.1.7:8080;
    server 192.168.1.8:8080;
}
```  
2. 在http/server节点下新增路径匹配和转发配置:   
```shell  
location /hello {
    proxy_pass http://hello;
}
```  

参考资料:  
[Nginx指令索引](http://www.howtocn.org/nginx:directiveindex)  