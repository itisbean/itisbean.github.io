---
title: Go爬虫 http code 521问题解决
date: 2020-03-29 23:48:56
tags:
    - Go goquery spider
categories:
    - Other
---

今天在帮朋友爬取好大夫网站（<i style="color:#ccc">只是获取某地区的简单数据，非商用</i>）的时候，一直被卡在获取医生个人页面上。打印错误信息发现返回的Http Status Code是521，但是直接打开网页又是正常的。对于这个状态码我并不熟悉，但是前功尽弃又实在不甘心，只好Google寻找答案。

<!-- more -->

最终是这篇文章帮到了我：[{% label info@HTTP STATUS CODE: 521的解决办法%}](https://blog.csdn.net/wangdepei/article/details/84798601)

我在postman和Chrome浏览器上还原这个场景，果然和文章描述非常一致。

[{% label info@List of HTTP status codes%}](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes) 中对http code 521的定义是：
> Web Server Is Down: The origin server has refused the connection from Cloudflare. 

实际上，这是网站防爬虫的一种措施。了解了这一措施的执行方式，也就可以抽丝剥茧一步一步去解决这个问题了。

具体的分析过程就不赘述了，直接参考上面那篇文章即可，这里我简单记录一下在代码中解决的完成过程：

1. 发起一个请求，这是正常操作，但是这一次请求肯定不能得到有效的页面，不过都是必走的过程，无法避免。

    (1) 先清除cookie（在postman下使用带cookie的请求得到的js代码无法解析，而浏览器是可以正常跳转的，这里不深究了，直接清掉以绝后患）;
        设置该请求连接为keep-alive（{% label danger@第二个请求要使用同一个req发起，所以这个请求是不能close的%}）

    ``` Go
    req.Header.Del("Cookie")
    req.Header.Add("Connection", "keep-alive")
    req.Close = false
    ```

    (2) 发起请求（略）

2. 获取响应头的Set-Cookie属性，打印出来该值如下：
    > {% label danger@ __jsluid_h=988846693f681c7e063343acefc11072%}; max-age=31536000; path=/; HttpOnly

    这里只需要拿到__jsluid_h的值就可以了

    ``` Go
    firstCookie := res.Header.Get("Set-Cookie")
    firstCookie = strings.Split(sc, ";")[0]
    ```

3. 获取response body，看起来像是一段js代码和一堆乱码组成，我们只需要{% label info@script标签之间的代码 %}，打印输出如下：

    ``` JavaScript
    var x="@var@rOm9XFMtA3QKV7nYsPGT4lifyWwkq5vcjH2IdxUoCbhERLaz81DNB6@while@parseInt@@29@@captcha@@replace@Expires@@1585492297@@split@37@RegExp@31@D@@f@length@0@firstChild@@catch@pathname@substr@36@window@cookie@@e@match@@__jsl_clearance@DOMContentLoaded@3@href@@else@0xEDB88320@String@charAt@addEventListener@https@new@false@g@Sun@2BU@challenge@toString@GMT@attachEvent@JgSe0upZ@@C@O@@@@search@location@0xFF@reverse@@@charCodeAt@@Mar@@@dt@Array@join@@try@@6@@eval@@K@chars@createElement@1@toLowerCase@return@@@iz7@@fromCharCode@1500@document@2@8@onreadystatechange@@if@@for@@20@@a@@@IDW@div@@69@setTimeout@Path@@function@15@@r@innerHTML@d@@@@@".replace(/@*$/,"").split("@"),y="2 2s=2m(){2j('1h.E=1h.s+1h.1g.b(/[\\?|&]9-15/,\\'\\')',20);21.w='B=e.2i|o|'+(2m(){2 1H=[2m(2s){1G 2s},2m(1H){1G 1H},(2m(){2 2s=21.1D('2g');2s.2q='<2c E=\\'/\\'>1w</2c>';2s=2s.p.E;2 1H=2s.z(/L?:\\/\\//)[o];2s=2s.t(1H.n).1F();1G 2m(1H){28(2 1w=o;1w<1H.n;1w++){1H[1w]=2s.J(1H[1w])};1G 1H.1t('')}})(),2m(2s){1G 1z('I.1L('+2s+')')}],1w=['1b',[[-~![]]+[-~![]]+(23+[]+[]),(23+[]+[])+(-~(+!![][{}])-~[-~![]-~![]]+[])],'1B',[(-~[-~![]-~![]]+[]+[])+[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]],'22',[[1x]+[1x]],'2f',({}+[]+[[]][o]).J((-~![]-~![])*[(-~![]-~![]<<-~![])]),'1c',[[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]],[[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]+[-~-~[]]],(!![][{}]+[]).J(D)+(-~(+!![][{}])-~[-~![]-~![]]+[]),[[1x]+(23+[]+[])],'1r',[[-~![]]+[-~![]]+(-~[-~![]-~![]]+[]+[]),[-~![]]+[-~-~[]]+[(+[])]],'2p',[(-~[-~![]-~![]]+[]+[])],[[1x]+(23+[]+[])],'1J',[(-~[-~![]-~![]]+[]+[])+[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]],'14%',(-~[-~![]-~![]]+[]+[]),'k'];28(2 2s=o;2s<1w.n;2s++){1w[2s]=1H[[1E,D,1E,D,1E,D,1E,o,1E,22,D,o,D,1E,D,1E,22,D,1E,D,1E,o,1E][2s]](1w[2s])};1G 1w.1t('')})()+';c=13, 7-1o-2a 2n:j:h 17;2k=/;'};26((2m(){1v{1G !!v.K;}r(y){1G 11;}})()){21.K('C',2s,11)}G{21.18('24',2s)}",f=function(x,y){var a=0,b=0,c=0;x=x.split("");y=y||99;while((a=x.shift())&&(b=a.charCodeAt(0)-77.5))c=(Math.abs(b)<13?(b+48.5):parseInt(a,36))+y*c;return c},z=f(y.match(/\w/g).sort(function(x,y){return f(x)-f(y)}).pop());while(z++)try{eval(y.replace(/\b\w+\b/g, function(y){return x[f(y,z)-1]||("_"+y)}));break}catch(_){}{% endnote %} 
    ```

4. 接下来要执行上面这段js代码，这里需要用到运行js代码的go包：[{% label info@github.com/robertkrimen/otto%}](https://github.com/robertkrimen/otto)，使用方法也很简单，直接参考文档就好。但是上面这段代码还是存在一些问题，直接运行是拿不到任何返回结果的。我的解决方式如下，先使用{% label info@console.log()%}输出控制台日志，再在js代码里加一段代码，返回最后一条log结果。代码如下：

    ``` Go
    //先把上段js代码中的eval替换成console.log
    js = strings.Replace(js, "eval(", " console.log(", -1)
    // 加入一段返回log的js代码
    consoleJs := "var lastLog = \"\";console.oldLog = console.log;console.log = function(str) {console.oldLog(str);lastLog = str;};"
    // 拼接起来，并执行最终这段js
    js = consoleJs + js
    ```

5. 执行后得到的第二段js代码如下，结果依然非常乱，其他的不用管，直接找到我们需要的第二个cookie的key，也就是{% label danger@__jsl_clearance%}，查看=后面的内容，1585492297.69|0| + function(){}，前半段很好说，主要是后半段的function返回的内容看起来十分隐晦，不过既然是js代码，应该还是可以通过前面的方法获取到的。
    <i style="font-size:12px;text-align:left;"> {% note default %} var _2s=function(){setTimeout('location.href=location.pathname+location.search.replace(/[\?|&]captcha-challenge/,\'\')',1500);==document.cookie='__jsl_clearance===1585492297.69|0|'+(function(){var _1H=[function(_2s){return _2s},function(_1H){return _1H},(function(){var _2s=document.createElement('div');_2s.innerHTML='<a href=\'/\'>_1w</a>';_2s=_2s.firstChild.href;var _1H=_2s.match(/https?:\/\//)[0];_2s=_2s.substr(_1H.length).toLowerCase();return function(_1H){for(var _1w=0;_1w<_1H.length;_1w++){_1H[_1w]=_2s.charAt(_1H[_1w])};return _1H.join('')}})(),function(_2s){return eval('String.fromCharCode('+_2s+')')}],_1w=['C',[[-~![]]+[-~![]]+(8+[]+[]),(8+[]+[])+(-~(+!![][{}])-~[-~![]-~![]]+[])],'K',[(-~[-~![]-~![]]+[]+[])+[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]],'2',[[6]+[6]],'IDW',({}+[]+[[]][0]).charAt((-~![]-~![])*[(-~![]-~![]<<-~![])]),'O',[[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]],[[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]+[-~-~[]]],(!![][{}]+[]).charAt(3)+(-~(+!![][{}])-~[-~![]-~![]]+[]),[[6]+(8+[]+[])],'dt',[[-~![]]+[-~![]]+(-~[-~![]-~![]]+[]+[]),[-~![]]+[-~-~[]]+[(+[])]],'r',[(-~[-~![]-~![]]+[]+[])],[[6]+(8+[]+[])],'iz7',[(-~[-~![]-~![]]+[]+[])+[(-~[]+[-~-~[]]>>-~-~[])-~(+!![][{}])-~[-~![]-~![]]]],'2BU%',(-~[-~![]-~![]]+[]+[]),'D'];for(var _2s=0;_2s<_1w.length;_2s++){_1w[_2s]=_1H[[1,3,1,3,1,3,1,0,1,2,3,0,3,1,3,1,2,3,1,3,1,0,1][_2s]](_1w[_2s])};return _1w.join('')})()+';Expires=Sun, 29-Mar-20 15:31:37 GMT;Path=/;'};if((function(){try{return !!window.addEventListener;}catch(e){return false;}})()){document.addEventListener('DOMContentLoaded',_2s,false)}else{document.attachEvent('onreadystatechange',_2s)} {% endnote %} </i> 

    以下就是截取出来的cookie值后半段的js代码了（因为在原代码段中这是个闭包函数，截取出来是无法直接执行的，所以只把function中的内容拿出来，并定义为一个新函数f。

    ``` JavaScript
    function f(){var _1H=[function(_2s){return _2s},function(_1H){return _1H},(function(){var _2s=document.createElement('div');_2s.innerHTML='<a href='/'>_1w';_2s=_2s.firstChild.href;var _1H=_2s.match(/https?:///)[0];_2s=_2s.substr(_1H.length).toLowerCase();return function(_1H){for(var _1w=0;_1w<_1H.length;_1w++){_1H[_1w]=_2s.charAt(_1H[_1w])};return _1H.join('')}})(),function(_2s){return eval('String.fromCharCode('+_2s+')')}],_1w=['C',[[-![]]+[-![]]+(8+[]+[]),(8+[]+[])+(-(+!![][{}])-[-![]-![]]+[])],'K',[(-[-![]-![]]+[]+[])+[(-[]+[--[]]>>--[])-(+!![][{}])-[-![]-![]]]],'2',[[6]+[6]],'IDW',({}+[]+[[]][0]).charAt((-![]-![])*[(-![]-![]<<-![])]),'O',[[(-[]+[--[]]>>--[])-(+!![][{}])-[-![]-![]]]],[[(-[]+[--[]]>>--[])-(+!![][{}])-[-![]-![]]]+[--[]]],(!![][{}]+[]).charAt(3)+(-(+!![][{}])-[-![]-![]]+[]),[[6]+(8+[]+[])],'dt',[[-![]]+[-![]]+(-[-![]-![]]+[]+[]),[-![]]+[--[]]+[(+[])]],'r',[(-[-![]-![]]+[]+[])],[[6]+(8+[]+[])],'iz7',[(-[-![]-![]]+[]+[])+[(-[]+[--[]]>>--[])-(+!![][{}])-[-![]-![]]]],'2BU%',(-[-![]-~![]]+[]+[]),'D'];for(var _2s=0;_2s<_1w.length;_2s++){_1w[_2s]=_1H[1,3,1,3,1,3,1,0,1,2,3,0,3,1,3,1,2,3,1,3,1,0,1][_2s]};return _1w.join('')}
    ```

    之后使用otto执行这个函数（或在js里执行，直接run获取结果也可），结果就是第二个cookie的后半段的值了。完整结果就是：{% label danger@__jsl_clearance=1585492297.69|0|CvTK%2BIDWOOHs4DdtqxrDiz7%2BU%3D%}

    补充：执行时可能还会遇到一些js错误，比如：
    {% label warning@'window' is not defined%}，{% label warning@'document' is not defined%} 等
    （因为不是在浏览器环境下，无法使用DOM对象）,在js里给些默认值优雅避过就好，例如：

    ``` Go
    docJs := "var window = {}; var document = {createElement:function(){ return {firstChild:{href:'https://'}, innerHTML:''}}};"
    ```

6. 最后，把完整的cookie加到请求头中再发起一起请求，就可以获得200的正常返回了。

    ``` JavaScript
    Cookie:__jsluid_h=988846693f681c7e063343acefc11072;__jsl_clearance=1585492297.69|0|CvTK%2BIDWOOHs4DdtqxrDiz7%2BU%3D
    ```
