title: 使用Nginx+Lua+Redis构建灰度发布环境  
date: 2015/3/7 20:00:00
tags: [nginx,lua,redis]
categories: [middleware]
---

灰度发布是指在黑与白之间, 能够平滑过渡的一种发布方式.
灰度发布可以保证应用系统的稳定, 降低产品升级影响的用户范围; 也可以按照一定的策略让部分用户提前参与产品测试, 从而提早获得用户意见反馈, 完善产品功能.

## 原理
使用nginx做负载均衡和反向代理, nginx内嵌lua模块, 解析并执行lua编写的脚本逻辑, 可以通过lua解析cookie以及访问redis, 而一些灰度分流的策略就是放在redis里通过cookie关联.
原理和[「利用nginx+lua+memcache实现灰度发布档」](http://www.cnblogs.com/wenbiao/p/3227998.html)这篇文章有点类似, 只是我们使用redis替换了memcache, 并且加入了cookie的识别, 比单纯的IP分流更强大一些.

## 环境搭建

- 安装lua解释器
  我这里用的是Luajit
  安装好以后需要配置环境变量, 编译nginx的时候会用到
- 下载lua-nginx-module, 编译安装nginx
- 下载安装lua redis library和lua cookie library

具体安装步骤可以参考下面的资料, 都很详细.
[CentOS安装nginx+nginx_lua模块](http://www.ttlsa.com/nginx/nginx-modules-ngx_lua/)
[Nginx+Lua+Redis构建高并发应用(Ubuntu)](http://www.ttlsa.com/nginx/nginx-lua-redis/)
[CentOS安装 nginx_lua_module 模块 以及 echo-nginx-module 模块](http://blog.csdn.net/vboy1010/article/details/7868645)


在网上查相关资料的时候还发现了一位大牛[YichunZhang](http://agentzh.org/), 也可以直接使用他提供的集成包[OpenResty](http://openresty.org/)快速安装.
他貌似也是[Nginx HttpLuaModule](http://wiki.nginx.org/HttpLuaModule)的作者.

## 配置和使用
在Nginx里使用Lua脚本操作Redis, 这篇文章介绍的比较详细[「Nginx + Lua + redis」](http://blog.csdn.net/vboy1010/article/details/7892120)
Lua语法其实也比较简单 [「Lua脚本语法说明」](http://www.cnblogs.com/ly4cn/archive/2006/08/04/467550.html)

Lua操作cookie和redis主要用到两个library:  
[lua cookie library](https://github.com/cloudflare/lua-resty-cookie)
[lua redis library](https://github.com/openresty/lua-resty-redis)

如果数据逻辑比较复杂还可以用json library:
[lua json library](https://github.com/craigmj/json4lua)

#### Lua redis library

- 支持pool
> Ensure you configure the connection pool size properly in the set_keepalive . Basically if your NGINX handle n concurrent requests and your NGINX has m workers, then the connection pool size should be configured as n/m. For example, if your NGINX usually handles 1000 concurrent requests and you have 10 NGINX workers, then the connection pool size should be 100.

- 不支持LB和sharding, 可以自己实现相关逻辑
  > You can trivially implement your own Redis load balancing logic yourself in Lua. Just keep a Lua table of all available Redis backend information (like host name and port numbers) and pick one server according to some rule (like round-robin or key-based hashing) from the Lua table at every request. You can keep track of the current rule state in your own Lua module's data, see http://wiki.nginx.org/HttpLuaModule#Data_Sharing_within_an_Nginx_Worker  

- 也可以引入[Codis](https://github.com/wandoulabs/codis), 参考[「豌豆荚分布式Redis的设计与实现」](http://www.infoq.com/cn/presentations/design-and-implementation-of-wandoujia-distributed-redis)

## 示例
Lua通过解析Cookie读取Redis数据进行分流

#### Lua脚本1


``` lua
	local redisLib = require "resty.redis"
	local cookieLib = require "resty.cookie"
	local json = require("json")

	local redis = redisLib:new()
	if not redis then
	    ngx.log(ngx.ERR, err)
	    ngx.exec("@defaultProxy")
	end

	local cookie, err = cookieLib:new()
	if not cookie then
	    ngx.log(ngx.ERR, err)
	    ngx.exec("@defaultProxy")
	end

	-- set cookie(模拟测试) 
	--[[
	local ok, err = cookie:set({
	    key = "uid", value = "100",
	})
	if not ok then
	    ngx.log(ngx.ERR, err)
	    ngx.exec("@defaultProxy")
	end
	]]

	-- get cookie
	local uid, err = cookie:get("uid")
	if not uid then
	    ngx.log(ngx.ERR, err)
	    ngx.exec("@defaultProxy")
	end

	redis:set_timeout(1000)
	local ok, err = redis:connect('127.0.0.1', '6379')
	if not ok then
	    ngx.log("failed to connect:", err)
	    ngx.exec("@defaultProxy")
	end

	-- 根据用户会话ID获取用户属性
	-- 也可以直接通过后端应用set到cookie然后在这里解析即可, 少一次redis调用
	-- eg: {'tag1':'2','tag2':'1','tag3':'0'}
	local tags, err = redis:get(uid)
	if not tags then
	        ngx.log("failed to get uid: ", err)
	        ngx.exec("@defaultProxy")
	end

	if tags == ngx.null then
	    ngx.log("uid not found.")
	    ngx.exec("@defaultProxy")
	end

	-- 获取规则配置信息, 需要做一定的缓存策略
	-- eg: {'tag':'tag1','proxy':{'0':'proxy_a','1':'proxy_a','2':'proxy_b'}}
	local proxyConfig, err = redis:get("proxyConfig")
	if not proxyConfig then
	        ngx.log("failed to get proxyConfig: ", err)
	        ngx.exec("@defaultProxy")
	end

	if proxyConfig == ngx.null then
	    ngx.log("proxyConfig not found.")
	    ngx.exec("@defaultProxy")
	end

	-- put it into the connection pool of size 100,
	-- with 10 seconds max idle time
	local ok, err = red:set_keepalive(10000, 100)
	if not ok then
	    ngx.say("failed to set keepalive: ", err)
	    return
	end


	proxyConfigData = json.decode(proxyConfig)
	tagsData = json.decode(tags)
	tag = proxyConfigData.tag

	-- 解析规则处理
	-- 根据规则里配置的类型和用户标签做匹配, 分流到相应的服务器上
	-- 这里是可以按照用户标签维度支持比较灵活的配置分流规则, 如果业务逻辑简单的话也可以简化
	proxy = "@defaultProxy"
	for k_tag, v_tag in pairs(tagsData) do
	        if k_tag == tag then
	                for k_proxy, v_proxy in pairs(proxyConfigData.proxy) do
	                        if v_tag == k_proxy then
	                                proxy = v_proxy
	                                break
	                        end
	                end
	        end
	end

	ngx.exec(proxy)

```

#### Nginx配置

- http配置
``` shell
lua_package_path "/home/wangq/work/nginx/lua/?.lua;;";

upstream proxy_a {
    #ip_hash;
    server 192.168.1.8:80;
    server 192.168.1.9:80;
}

upstream proxy_b {
    #ip_hash;
    server 192.168.1.10:80;
    server 192.168.1.11:80;
}
```

- server配置
``` shell
location /lua {
            content_by_lua_file /home/wangq/work/nginx/lua/hello.lua;
    }

    location @defaultProxy {
            proxy_pass http://proxy_a;
    }

    location @proxy_a {
            proxy_pass http://proxy_a;
    }

    location @proxy_b {
            proxy_pass http://proxy_b;
    }

```

## Troubleshooting
如果发现配置不生效可以查看nginx的error log, lua脚本异常可以根据堆栈信息定位分析.

类似下面的异常信息, 是因为nginx执行用户没有读权限, nginx默认应该是用nobody用户启动的
> failed to load external Lua file "xxx.lua": cannot open xxx.lua: Permission denied

