title: Hexo  
date: 2014/7/20 20:46:25
updated: 2014/7/25 23:00:00
tags: [hexo,node.js,blog]
categories: [hexo]
---

## 为什么选Hexo  

主要是以下几个原因:  
- **MarkDown**  
  易读易写, 不用考虑页面样式.  
- **Node.js**  
  曾经用过一段时间Node.js, 有点经验, 如果有问题还可以自己debug, 这也是我为什么没有考虑 **Jekyll** 或者 **Octopress**的原因.  
- **GitHub Pages**  
  可以很容易部署托管到GitHub上, 另外也可以支持GitCafe.  

关于blog产品比较, 知乎上有个相关问题也可以参考下:  
[这些博客程序有什么特点](http://www.zhihu.com/question/21981094)  

我在用Hexo之前也有试过Farbox和Ghost, 说下我的试用感受:  
- Farbox基于DropBox, 安装比较简单, 也比较好用, 不过是要收费的, 比较适合不喜欢折腾的人用, 如果后来没有找到Hexo的话我应该也会选这个.  
- Ghost也是基于Node.js开发, Markdwon写作, 不过真心觉得不好用.  


## Hexo安装  

安装其实也比较简单, 可以参考官方文档或者以下资料, 都很详细.  
[简明Github Pages与Hexo教程](http://cnfeat.com/2014/05/10/2014-05-11-how-to-build-a-blog/)  
[搭建hexo博客](http://zipperary.com/2013/05/28/hexo-guide-2/)  

另外我在搭建过程中碰到了几个问题, 后面也会记录一下.  


## 安装过程中碰到的问题

### 页面首次加载时很卡  

是因为页面样式里有默认引用了googleapi的相关js和样式文件, 而googleapi因为某些 *你懂的* 的原因导致加载很慢甚至无法加载, 可以用360的google公共库代替.  
顺便这里给360点个赞, 终于干了件好事... :-)  
**解决办法:** 找到以下两个文件,   
themes\landscape\layout\_partial\head.ejs  
themes\landscape\layout\_partial\after-footer.ejs  
搜索 **googleapis** 替换为 **useso** 即可.  

[360前端公共库](http://libs.useso.com/)  


### 部署时提示  **" bad indentation of a mapping entry"**的异常信息  

**解决办法:** 检查一下配置文件里的配置项, key和value中间的分号之后要有空格.  


### 部署时提示  **"spawn ENOENT"**的异常信息  

> [error] Error: spawn ENOENT  
>  Error: spawn ENOENT  
>  at errnoException (child_process.js:1000:11)  
>  at Process.ChildProcess._handle.onexit (child_process.js:791:34)  


**解决办法:** 在命令行里输入git, 回车, 如果无法识别则需要把git安装目录下的bin目录文件路径加到环境变量path中.  


### 部署时提示  **"Not a git repository"**的异常信息  

> fatal: Not a git repository (or any of the parent directories): .git  


**解决办法:** 删掉.deploy目录试试.  


### 配置二级域名CNAME 不生效  

**解决办法:** 在项目的根目录下新建一个名为CNAME的文件, 文件内容就是二级域名.
eg: blog.pingan.im  


## 问题分析步骤  

1. 首先根据错误信息上网搜一下看这个问题别人有没有碰到过, 是不是有解决方案可以参考.  
2. 如果网上找不到解决办法, 那就根据异常信息去Hexo应用文件目录里批量搜一下, 一般都可以搜索到, 然后在抛异常的前面打印相关信息进行Debug分析.  