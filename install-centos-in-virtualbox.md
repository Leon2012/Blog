title: VirtualBox上安装CentOS
date: 2014/8/9 20:00:00
tags: [linux]
categories: [linux]
---

在VirtualBox上安装CentOS, 网络上有很多教程, 本身也比较简单, 我这里只是记一下安装碰到的问题:  

### 进不了图形界面  
其实一般情况下也不用图形界面, 主要是因为之前用相同的安装文件装过是没有问题的, 结果这次各种重试都不行.  
上网搜了下应该是内存太小的原因, 我之前设置的VM内存是1G, 这次用的是默认的512M.  

后来重试的时候发现其实是有一行提示的没注意看到  

> You do not have enough RAN to use the graphical installer. Starting text mode.  

** 解决办法: **  
调整VM内存即可.  
官网其实也有[说明](http://wiki.centos.org/Manuals/ReleaseNotes/CentOS6.5)的  
> The installer needs at least 406MB of memory to work. Text mode will automatically be used if the system has less than 632MB of memory.  

### 挂载共享目录  
参考[这里](http://jingyan.baidu.com/article/2fb0ba40541a5900f2ec5f07.html)配置.  

- 如果安装增强功能时提示错误:  
> Error: unable to find the sources of your current Linux kernel. Specify KERN_DIR=<directory> and run Make again.  

  ** 解决办法: ** [安装内核源文件](http://my.oschina.net/mkh/blog/225501).  

- 开机自动挂载:  
  修改/etc/fstab的方法不起作用, 在/etc/rc.local中添加mount命令, 启动以后会自动执行挂载.  
  参考: [VirtualBox共享文件夹设置及开机自动挂载](http://www.cnblogs.com/52linux/archive/2012/03/07/2384381.html)

顺便发现/etc/rc.local是个好东东, 开机以后自动调用, 比较强大.  


### 网络设置  
连接方式要选择** 桥接网卡 **  
