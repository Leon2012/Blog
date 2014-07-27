title: Java性能问题分析及调优总结    
date: 2014/7/25 20:00:00  
tags: [java,jvm]  
categories: [java]  
---  
  
## thread dump分析    
  - 用 **kill -3 pid** 命令生成thread dump文件  
  - 可以用 **threadlogic** 工具导入生成的dump文件进行分析  
  - 参考资源:  
    - [三个实例演示](http://www.cnblogs.com/zhengyun_ustc/archive/2013/01/06/dumpanalysis.html)  
    - [Tomcat thread dump分析](http://www.jiacheo.org/blog/279)  
  
## heap dump分析  
  - 用jmap命令生成dump文件:   
    ``` java  
    jmap -dump:format=b,file=/tmp/logs/dump.bin pid   
    ```
  - jmap命令会 **stop the world** , 生产环境需谨慎处理  
  - 安装 **MAT** 工具,导入dump文件进行分析  
  - 可以在linux环境下分析, 生成网页格式的报告拿回win环境下查看  
  - 参考资源:  
    - [MAT](http://www.eclipse.org/mat)  
  
## cpu load average too high  
  1. 用 **top** 命令找到cpu占比高的java进程pid  
  2. 用 **top -Hp pid** 命令找到cpu占比高的线程pid  
  3. 看到的线程pid是十进制的, 转成十六进制(可以用windows计算器转换, 查看->程序员->十六进制)   
  4. 用 **kill -3 pid** 命令打thread dump  
  5. 在dump文件中查找第三步得到的十六进制字符串对应的pid, 然后根据堆栈信息进行分析  
    
## gc log分析  
  - 修改jvm参数打印gc log  
    ``` java  
    -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/path/logs/jvm.log  
    ```  
  - 用 **GCLogViewer** 工具导入gc log进行分析  
  
## JVM参数调优   
### Heap堆参数设置   
  - -Xmx: JVM最大可用内存   
  - -Xms: JVM初始内存, 一般设置与-Xmx相同, 以避免每次垃圾回收完后JVM重新分配内存.  
  - -Xmn: sun官方推荐年轻代配置为整个堆的3/8  
  - -Xss: 设置每个线程的堆栈大小.在相同物理内存下, 减小这个值能生成更多的线程.但是操作系统对一个进程内的线程数还是有限制的, 不能无限生成, 经验值在3000~5000左右.  
  - -XX:NewRatio=4: 设置年轻代（包括Eden和两个Survivor区）与年老代的比值（除去持久代）.设置为4, 则年轻代与年老代所占比值为1:4, 年轻代占整个堆栈的1/5  
  - -XX:SurvivorRatio=4: 设置年轻代中Eden区与Survivor区的大小比值.设置为4, 则两个Survivor区与一个Eden区的比值为2:4, 一个Survivor区占整个年轻代的1/6  
  - -XX:MaxPermSize=16m: 设置持久代大小为16m.  
  - -XX:MaxTenuringThreshold=0: 设置垃圾最大年龄.如果设置为0的话, 则年轻代对象不经过Survivor区, 直接进入年老代.对于年老代比较多的应用, 可以提高效率. 如果将此值设置为一个较大值, 则年轻代对象会在Survivor区进行多次复制, 这样可以增加对象再年轻代的存活时间,增加在年轻代即被回收的概论.   
    
### GC Collections回收器类型  
  - -XX:+UseSerialGC: 设置串行收集器  
  - -XX:+UseParallelGC: 设置并行收集器  
  - -XX:+UseParalledlOldGC: 设置并行年老代收集器  
  - -XX:+UseConcMarkSweepGC: 设置并发收集器  
  
### 调优准则  
  - 活跃数据大小: 应用程序运行稳定态时长期存活对象在Java堆中占用的空间大小, 即Full GC之后Java堆中老年代和永久代占用的空间  
  - 老年代空间大小不应该小于活跃数据大小的1.5倍  
  - 新生代空间至少应为Java堆大小的10%, 过小可能导致频繁Minor GC  
  - 增大Java堆大小不要超过JVM可用的物理内存数  
  - -XX:MaxTenuringThreshold=n 晋升阀值不建议设置为0, 因为会导致新创建的对象在Minor GC一次以后直接进入老生代, 导致老生代频繁Full GC; 也不建议大于实际可能的最大值, 会造成对象长期存在Survivor空间直到溢出, 一旦溢出对象会被全部提升至老生代.可以通过 **-XX:+PrintTenuringDistribution** 打印日志监控以确定最优值.  
  - 调整Survivor空间容量时, 如果新生代空间不变, 增大Survivor空间会减少Eden空间从而增加Minor GC频率；如果要保持Minor GC频率则保持Eden空间大小不变, 增加新生代空间.  
  - 调优CMS停顿时间:   
    - -XX:ParallelGCThreads=n: 并行收集的线程数, 最好配置于处理器数目相等  
    - –XX:CMSWaitDuration: 设置收集暂停的最大间隔时间  
    - -XX:+CMSParallelRemarkEnabled: 开启并行Remark, 降低第二次暂停时间  
    - -XX:+CMSScavengeBeforeRemark: 强制进入CMS重新标记阶段之前先进行一次Minor GC, 减少remark暂停时间, 但是在remark之后也将立即开始有一次Minor GC  
    - -XX:+CMSPermGenSweepingEnabled -XX:+CMSClassUnloadingEnabled: 开启CMS回收持久代  
    - -XX:CMSInitiatingOccupancyFraction=80: 默认老生代占满68%时开始CMS GC, 通过这个配置项可以降低CMS GC次数  
  
### java堆大小计算法则  
  - Java堆: 3~4倍Full GC后老年代空间占用量  
  - 永久代: 1.2~1.5倍Full GC后永久代空间占用量  
  - 新生代: 1~1.5倍Full GC后老年代空间占用量  
  - 老生代: 2~3倍Full GC后老年代空间占用量  
  
### 其他  
  - 日志分析  
    ``` java  
    -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/path/logs/jvm.log  
    -XX:+HeapDumpOnOutOfMemoryError  
    ```
  - Minor GC、Major GC(Full GC)  
  - Full GC: Stop The World  
  - 参考资源  
    - [JVM调优总结](http://unixboy.iteye.com/blog/174173)  
    - [深入浅出Java垃圾回收机制](http://my.oschina.net/xishuixixia/blog/131212)  
    - [Java 6 JVM参数选项大全](http://kenwublog.com/docs/java6-jvm-options-chinese-edition.htm)  
    - [Java GC CMS 日志分析](http://blog.csdn.net/nini_lou/article/details/23207399)  
    - [探秘Java虚拟机——内存管理与垃圾回收](http://www.blogjava.net/chhbjh/archive/2012/01/28/368936.html)  
    - [记一次JVM GC日志分析](http://blog.csdn.net/cpzhong/article/details/6831751)  
    - [CMS GC实践总结](http://www.blogjava.net/killme2008/archive/2009/09/22/295931.html)  
