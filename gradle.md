title: 使用gradle管理java依赖包
date: 2015/10/06 20:00:00
tags: [gradle,java]
categories: opensource
---

### 下载与安装
参考[gradle user guide](https://docs.gradle.org/current/userguide/installation.html)

### 修改gradle本地仓库缓存目录
默认的缓存目录是在用户HOME目录下的.gradle文件夹, 例如c:/users/admin/.gradle/  
假如需要修改到D盘目录, 步骤如下:
1. cp c:/users/admin/.gradle/ d:/.gradle/  
2. 新增环境变量GRADLE_USER_HOME=d:/.gradle/

### 配置gradle代理
如果因为一些原因(例如公司网络管控)无法直接访问仓库地址, 可以通过配置http代理的方式访问.
在gradle默认缓存目录下, 新增配置文件gradle.properties  
``` gradle
systemProp.http.proxyHost=Proxy Server
systemProp.http.proxyPort=Proxy port
systemProp.http.proxyUser=Proxy User
systemProp.http.proxyPassword=Proxy Password
systemProp.http.nonProxyHosts=*.nonproxyrepos.com|localhost

systemProp.https.proxyHost=Proxy Server
systemProp.https.proxyPort=Proxy port
systemProp.https.proxyUser=Proxy User
systemProp.https.proxyPassword=Proxy Password
systemProp.https.nonProxyHosts=*.nonproxyrepos.com|localhost
```
上面两组配置看起来貌似一样吧, 但仔细看下就会发现下面是 **https** 的配置哦(本人当时就被坑了一把...)

### 管理java依赖包

然后就是配置啦, gradle本身配置就很简单, 所以很容易看懂.
```
apply plugin: 'java'

repositories {
    mavenCentral()
}

dependencies {
    def springVersion = "3.2.4.RELEASE"
    compile 'org.springframework:spring-core:${springVersion}',
            'org.springframework:spring-aop:${springVersion}'
    compile group: 'commons-collections', name: 'commons-collections', version: '3.2'
    testCompile group: 'junit', name: 'junit', version: '4.+'
}

task copyJars(type: Copy) {
    from configurations.runtime
    into 'lib'
}

task listJars << {
    configurations.compile.each { File file -> println file.name }
}
```
最后就是验证啦:   
`gradle -q copyJars` 下载jar包到指定的lib目录  
`gradle -q listJars` 打印jar包列表

虽然看起来比较简单, 但是gradle功能还是很强大的, 详细的依赖管理介绍参考[Gradle Dependency Management](https://docs.gradle.org/current/userguide/dependency_management.html).
