---
layout: post
title: 在Windows Server 2012上的IIS8.5中配置伪静态
author: 宋强
tags: IIS Web
---

这个网站本身是一个asp.net网站，参数都是url传参，但是由于一些原因百度蜘蛛不会收录小网站的带有问好参数的网页，所以如果能够对网页url进行伪静态重写的话就可以让百度放心的收录我们的网页。

![](../../../images/URL&#32;Rewrite/Server.jpg)

查看是否存在Url Rewriter选项

![](../../../images/URL&#32;Rewrite/Rewrite&#32;Icon.jpg)

如果不存在的话需要安装这个组件，可以在下面Windows的官方网站链接下载：

[URL Rewrite : The Official Microsoft IIS Site](https://www.microsoft.com/web/handlers/webpi.ashx/getinstaller/urlrewrite2.appids)

下载安装之后重启IIS就有这个选项了，现在我们开始正式设置URL重写
示例是将形如

article/12

的URL转换为

article/article.aspx?articleId=12

# 配置规则
打开Url Rewriter，单击右边的添加规则，之后选择空白规则：

![](../../../images/URL&#32;Rewrite/Create&#32;rule.jpg)

在新出现的窗口中，名称随便起，之后匹配模式这里我们用正则表达式（不熟悉的小朋友快回去补啊）

![](../../../images/URL&#32;Rewrite/Edit&#32;rule.jpg)

点击测试模式，输入一个示例进行测试，得到我们的参数都由哪些变量表示

![](../../../images/URL&#32;Rewrite/Regex.jpg)

再到最下面找到操作（Action），在下面的重写URL中写入自己想要转到的URL，在这里也就是

article/article.aspx?articleId={R:1}

![](../../../images/URL&#32;Rewrite/Regex&#32;target.jpg)

再单击右上角的应用就配置完成了，大功告成！

# 参考
1. [Creating Rewrite Rules for the URL Rewrite Module](https://www.iis.net/learn/extensions/url-rewrite-module/creating-rewrite-rules-for-the-url-rewrite-module)