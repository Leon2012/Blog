title: nodejs rename error  
date: 2014/10/07 20:00:00
tags: [nodejs]
categories: [nodejs]
---

有一年多没用过nodejs了, 这两天心血来潮想写个小东西突然发现都忘的差不多了, 好忧桑...  
最近写代码的能力直线下降, 写代码的机会也越来越少, 会不会哪一天java代码都不会写了...  


还是先记一下折腾nodejs碰到的问题吧:  

- rename的时候提示错误: Error: EXDEV  
	参考资料: [Renaming files cannot be done cross-device](http://stackoverflow.com/questions/21071303/node-js-exdev-rename-error)
	windows在不同盘符下rename时会报这个错.
	我碰到的一个场景是文件上传的时候, 通常情况下windows系统临时目录在c盘. 上传文件的时候会自动存到临时目录, 这个时候如果在其他盘下执行nodejs rename就会报这个错.  
	**解决办法:** 修改上传临时目录或者目标路径到同一个盘符即可.  


- 处理文件的时候提示错误: Error: ENOENT  
  检查文件路径是否正确, 是否上级目录还没有创建.  
	