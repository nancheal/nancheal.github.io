---
layout: post
title: sqlmapTricks
date:   2018-07-13 19:42:45 +0800
categories: [sqlmap, tricks]
---

> 记录一些工作中经常使用的sqlmap的tricks

### sqlmap 探测技巧

- sqlmap -v ; 一般都会将这个值调大来查看sqlmap到底在发送什么样的数据包
- sqlmap -d ; 直接连接数据库，这个需要去安装第三方的依赖，docker4sqlmap中已经安装
- sqlmap --risk ; 越大发送的数据包越多，当risk=3时会发送or语句，可能会影响业务，线上环境慎用
- sqlmap --level ; 越大发送的数据包也就越多
- sqlmap --technique ；BEUS 分别对应不同的类型的注入
- sqlmap --dbms ; 指定数据库的类型

### sqlmap 绕过技巧

- sqlmap --csrf-token、--csrf-url ; 指定csrf的参数名/获取csrf参数的url，这种方法往往用以绕过csrf的保护进行注入，但是这种方法只能指定一个参数，对于多个参数无法使用
- sqlmap --eval ; 用以在sqlmap发包前执行的脚本，这种方法往往用来替换数据包中的某些参数，所以这种方法可以用来绕过多个或者单个参数的csrf防护，这里在编写自定义脚本时可能需要去注意：由于脚本是拿双引号括起来的，所以脚本中涉及到的一些字符串可能需要使用单引号括起来，还有python用于分割每一条语句的分号，如果出现在其他非分割语句的位置也需要进行转义后才可以使用，大多数脚本报错问题都可以上面两个思路去找报错的原因
- sqlmap --eval ; 从上面的tricks我们知道sqlmap可以通过--eval来替换一些变量，那么如果我们想替换http消息头中的一些内容我们应该怎么办？--eval="_locals['auxHeaders']['Host'] = 'foo.com'" -v 5 ，通过这样的方式来替换http头，sqlmap中一些其他有用的变量uri（地址）、lastPage（应该是上一个请求的响应）、_locals（一些有用的变量）
- random-agent ; 使用这个参数来隐藏sqlmap自带的请求头，以规避检查
### sqlmap 利用部分

- need to do