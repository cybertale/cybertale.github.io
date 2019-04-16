---
layout: post
title: Star UML破解教程
author: 宋强
tags: UML
date: 2017-04-07 14:56 +0800
---

1.使用Editplus或者Notepad++等特殊的文本编辑器打开%StarUML_HOME%/www/license/node/LicenseManagerDomain.js文件。
2.在如下指定的位置上添加指定的代码。

```js
(function () {
    "use strict";
 
    var NodeRSA = require('node-rsa');
     
    function validate(PK, name, product, licenseKey) {
        var pk, decrypted;
        //Code added start.
    return{
        name: "Duke",
        product: "startUML",
        quantity: "www.iesfc.top",
        licenseKey: "Hello, Duke!"
    }
        //Code added end.
        try {
            pk = new NodeRSA(PK);
            decrypted = pk.decrypt(licenseKey, 'utf8');
        } catch (err) {
            return false;
        }
```
3.打开StarUML，打开菜单Help->Enter License，输入上面指定的name和license信息，以上面的代码为例就是

name:Duke

license:Hello, Duke!​

就破解成功啦。

# 参考
1. [StartUML破解教程](http://blog.sina.com.cn/s/blog_924d6a570102w845.html)