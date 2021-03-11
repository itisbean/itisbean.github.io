---
title: Go爬虫 http code 521问题解决
date: 2020-03-29 23:48:56
tags:
    - Go 
    - 网络爬虫
    - 实战总结
categories:
    - Coding
---

今天在帮朋友爬取好大夫网站的时候，一直被卡在请求医生个人主页上。打印错误信息发现返回的http code都是521，而直接打开网页又是正常的。对于这个状态码我并不了解，但是前功尽弃又实在不甘心，只好Google寻找答案。

<!-- more -->

最终是这篇文章帮到了我：[{% label info@HTTP STATUS CODE: 521的解决办法%}](https://blog.csdn.net/wangdepei/article/details/84798601)

我在Postman和Chrome浏览器上还原这个场景，果然和文章描述非常一致。

[{% label info@List of HTTP status codes%}](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes) 中对http code 521的定义是：
> Web Server Is Down: The origin server has refused the connection from Cloudflare.

实际上，这是网站防爬虫的一种措施。了解了这一措施的执行方式，也就可以抽丝剥茧一步一步去解决这个问题了。

具体的分析过程就不赘述了，直接参考上面那篇文章即可，这里我简单记录一下在代码中解决的完整过程：

1. 发起一个请求。这是正常操作，但是这一次请求肯定不能得到有效的页面，不过都是必走的过程，无法避免。
    (1) 先清除cookie（在postman下使用带cookie的请求得到的js代码无法解析，而浏览器是可以正常跳转的，这里不深究了，直接清掉以绝后患）;

    ``` Go
    req.Header.Del("Cookie")
    ```

    (2) 设置该请求的Connection为keep-alive（{% label danger@第二个请求要使用同一个req发起，所以这个请求是不能close掉的%}）

    ``` Go
    req.Header.Add("Connection", "keep-alive")
    req.Close = false
    ```

    (3) 发起请求并拿到响应结果（略）

2. 获取响应头的Set-Cookie属性，打印出来该值如下，这里只需要拿到__jsluid_h的值即可：

    ``` JavaScript
    __jsluid_h=a44718981efe4f7006fece7f82d69844; max-age=31536000; path=/; HttpOnly
    ```

3. 获取response body，看起来像是一段js代码和一堆乱码组成，我们只需要`<script>`和`</script>`之间的js代码 ，打印输出如下：
    <i style="font-size:12px;"> {% note default %}
    var x="captcha@charAt@@@toLowerCase@@@onreadystatechange@Mar@Expires@firstChild@@@@chars@innerHTML@40@parseInt@if@while@@@createElement@@H@String@0xEDB88320@join@fromCharCode@@30@a@@@div@20@match@Mon@addEventListener@@DOMContentLoaded@split@17@19@pathname@@@challenge@rOm9XFMtA3QKV7nYsPGT4lifyWwkq5vcjH2IdxUoCbhERLaz81DNB6@Path@else@GMT@href@toString@e@function@@0xFF@false@@0@RegExp@@catch@@JgSe0upZ@Array@@@27@@@@__jsl_clearance@3@@substr@3D@document@@@return@k7zgL@location@g@cookie@@replace@window@2@1500@h@67Yn@@eval@new@search@@8@try@1585586419@@6@@charCodeAt@@reverse@f@d@var@@attachEvent@setTimeout@@@n@https@for@@@@length@@36@1@@@2Fs".replace(/@*$/,"").split("@"),y="2u o=1g(){2x('24.1d=24.15+24.2h.28(/[\\?|&]1-18/,\\'\\')',2b);1D.26='1y=2l.1u|1l|'+(1g(){2u y=[1g(o){22 o},1g(y){22 y},(1g(){2u o=1D.n('z');o.g='<w 1d=\\'/\\'>7</w>';o=o.b.1d;2u y=o.B(/2B?:\\/\\//)[1l];o=o.1B(y.32).5();22 1g(y){2C(2u 7=1l;7<y.32;7++){y[7]=o.2(y[7])};22 y.s('')}})(),1g(o){22 2f('q.t('+o+')')}],7=[[(-~[-~![]-~![]]+[]+[])+[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]],[-~-~[]],[[2n]+[2n],[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]+[(+[])]],'2c',[[2n]+(-~(2j)+[])],'2A',[(2j+[]+[])+(2j+[]+[])],[2n]+((-~(+!![][{}])+[~~{}])/[(-~![]<<-~![])]+[[]][1l]),'p%38',[-~-~[]],[[-~![]]+[-~![]]+(-~[-~![]-~![]]+[]+[])],'2d',[[-~![]]],'23',[(-~[-~![]-~![]]+[]+[]),((-~(+!![][{}])+[~~{}])/[(-~![]<<-~![])]+[[]][1l])],(!-{}+[]).2(~~[]),'25',[(-~[-~![]-~![]]+[]+[])+[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]],'1C'];2C(2u o=1l;o<7.32;o++){7[o]=y[[1z,1l,1z,35,1z,35,1z,1l,35,1l,1z,35,2a,35,2a,1l,35,1z,35][o]](7[o])};22 7.s('')})()+';a=C, v-9-A 13:h:14 1c;1a=/;'};j((1g(){2k{22 !!29.D;}1o(1f){22 1j;}})()){1D.D('11',o,1j)}1b{1D.2w('8',o)}",f=function(x,y){var a=0,b=0,c=0;x=x.split("");y=y||99;while((a=x.shift())&&(b=a.charCodeAt(0)-77.5))c=(Math.abs(b)<13?(b+48.5):parseInt(a,36))+y*c;return c},z=f(y.match(/\w/g).sort(function(x,y){return f(x)-f(y)}).pop());while(z++)try{ console.log(y.replace(/\b\w+\b/g, function(y){return x[f(y,z)-1]||("_"+y)}));break}catch(_){}{% endnote %}</i>

4. 接下来要执行上面这段js，这里需要用到运行js代码的go包：[{% label info@github.com/robertkrimen/otto%}](https://github.com/robertkrimen/otto)，使用方法也很简单，直接参考文档就好。但是上面这段代码还是存在一些问题，直接运行是拿不到任何返回结果的。我的解决方式如下，先使用{% label primary@console.log()%}输出控制台日志，再在js代码里加一段代码，返回最后一条log结果。代码如下：

    ``` Go
    //先把上段js代码中的eval替换成console.log
    js = strings.Replace(js, "eval(", " console.log(", -1)
    // 加入一段返回log的js代码
    consoleJs := "var lastLog = \"\";console.oldLog = console.log;console.log = function(str) {console.oldLog(str);lastLog = str;};"
    // 拼接起来，并执行最终这段js
    js = consoleJs + js
    ```

5. 执行后得到的第二段js代码如下，结果依然非常乱，其他的不用管，直接找到我们需要的第二个cookie的key，也就是{% label danger@__jsl_clearance%}，=后面的内容就是我们需要的值：1585492297.69|0| + function(){}，前半段很好说，主要是后半段的function的内容看起来十分隐晦，不过既然是js代码，我们还是可以通过运行它来获取答案的。

    <i style="font-size:12px"> {% note default %}
    var _o=function(){setTimeout('location.href=location.pathname+location.search.replace(/[\?|&]captcha-challenge/,\'\')',1500);{% label danger@document.cookie='__jsl_clearance=1585586419.27|0|'%}+(function(){var _y=[function(_o){return _o},function(_y){return _y},(function(){var _o=document.createElement('div');_o.innerHTML='<a href=\'/\'>_7</a>';_o=_o.firstChild.href;var _y=_o.match(/https?:\/\//)[0];_o=_o.substr(_y.length).toLowerCase();return function(_y){for(var _7=0;_7<_y.length;_7++){_y[_7]=_o.charAt(_y[_7])};return _y.join('')}})(),function(_o){return eval('String.fromCharCode('+_o+')')}],_7=[[(-~[-~![]-~![]]+[]+[])+[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]],[-~-~[]],[[6]+[6],[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]+[(+[])]],'h',[[6]+(-~(8)+[])],'n',[(8+[]+[])+(8+[]+[])],[6]+((-~(+!![][{}])+[~~{}])/[(-~![]<<-~![])]+[[]][0]),'H%2Fs',[-~-~[]],[[-~![]]+[-~![]]+(-~[-~![]-~![]]+[]+[])],'67Yn',[[-~![]]],'k7zgL',[(-~[-~![]-~![]]+[]+[]),((-~(+!![][{}])+[~~{}])/[(-~![]<<-~![])]+[[]][0])],(!-{}+[]).charAt(~~[]),'g',[(-~[-~![]-~![]]+[]+[])+[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]],'3D'];for(var _o=0;_o<_7.length;_o++){_7[_o]=_y[[3,0,3,1,3,1,3,0,1,0,3,1,2,1,2,0,1,3,1][_o]](_7[_o])};return _7.join('')})()+';Expires=Mon, 30-Mar-20 17:40:19 GMT;Path=/;'};if((function(){try{return !!window.addEventListener;}catch(e){return false;}})()){document.addEventListener('DOMContentLoaded',_o,false)}else{document.attachEvent('onreadystatechange',_o)}
    {% endnote %} </i>

    以下就是截取出来的cookie值后半段的js代码了（因为在原代码段中这是个闭包函数，截取出来是无法直接执行的，所以把function中的内容单独拿出来，定义为一个新函数f。

    <i style="font-size:12px"> {% note default %}
    function f(){var _y=[function(_o){return _o},function(_y){return _y},(function(){var _o=document.createElement('div');_o.innerHTML='<a href=\'/\'>_7</a>';_o=_o.firstChild.href;var _y=_o.match(/https?:\/\//)[0];_o=_o.substr(_y.length).toLowerCase();return function(_y){for(var _7=0;_7<_y.length;_7++){_y[_7]=_o.charAt(_y[_7])};return _y.join('')}})(),function(_o){return eval('String.fromCharCode('+_o+')')}],_7=[[(-~[-~![]-~![]]+[]+[])+[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]],[-~-~[]],[[6]+[6],[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]+[(+[])]],'h',[[6]+(-~(8)+[])],'n',[(8+[]+[])+(8+[]+[])],[6]+((-~(+!![][{}])+[~~{}])/[(-~![]<<-~![])]+[[]][0]),'H%2Fs',[-~-~[]],[[-~![]]+[-~![]]+(-~[-~![]-~![]]+[]+[])],'67Yn',[[-~![]]],'k7zgL',[(-~[-~![]-~![]]+[]+[]),((-~(+!![][{}])+[~~{}])/[(-~![]<<-~![])]+[[]][0])],(!-{}+[]).charAt(~~[]),'g',[(-~[-~![]-~![]]+[]+[])+[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]],'3D'];for(var _o=0;_o<_7.length;_o++){_7[_o]=_y[[3,0,3,1,3,1,3,0,1,0,3,1,2,1,2,0,1,3,1][_o]](_7[_o])};return _7.join('')} {% endnote %} </i>

    尝试执行这段js，问题又出现了。开始报{% label warning@'document' is not defined%} 等错误。报错主要集中在这一部分中：{% label danger@ var _o=document.createElement('div');_o.innerHTML='<a href=\'/\'>_7</a>';_o=_o.firstChild.href;var _y=_o.match(/https?:\/\//)[0];_o=_o.substr(_y.length).toLowerCase();%}，无论你熟不熟悉js都应该看得出这段看似冗长的代码实际上不过是在给一个变量赋值。
    但是这些错误很难破解，因为{% label primary@不是在浏览器环境下，无法使用DOM对象%}。不过都想到这里了，那不妨试试在浏览器下执行它会是什么结果。打开Chrome控制台执行这部分代码，并输出最终的这个变量值。
    看到结果后我不禁长舒一口气，竟然是当前页面的域名。既然如此那就不用绞尽脑汁去想如何解决js报错了，直接{% label primary@将这个变量赋值为当前域名%}就好了。
    之后使用otto执行这个函数（或在js里执行，直接run()获取结果都可）。这样就拿到第二个cookie的后半段的值了。和前半部分拼接起来就是我们要的答案：
    
    ``` JavaScript
    __jsl_clearance=1585492297.69|0|CvTK%2BIDWOOHs4DdtqxrDiz7%2BU%3D
    ```

    补充：如果还遇到类似{% label warning@'window' is not defined%}的报错，直接在前面创建一个默认对象就好，如下

    ``` Go
    preJs := "var window = {};"
    ```

6. 最后，把完整的cookie加到请求头中再发起一起请求，就可以获得200的正常返回了。

    {% note success %}
    Cookie: __jsluid_h=a44718981efe4f7006fece7f82d69844;__jsl_clearance=1585586419.27|0|%2BFhEnX65H%2Fs2q67Ynuk7zgLietg%3D
    {% endnote %}
