原生JS实现AJAX、JSONP及DOM加载完成事件，并提供对应方法

<!-- more -->
本文主要介绍了利用原生Javascript实现了异步加载，及跨域请求JSONP的实现方式。

## 一、JS原生AJAX

ajax：一种请求数据的方式，不需要刷新整个页面；
ajax的技术核心是 XMLHttpRequest 对象；
ajax 请求过程：创建 XMLHttpRequest 对象、连接服务器、发送请求、接收响应数据；

下面简单封装一个函数，之后稍作解释

``` javascript
	ajax({
        url: "./TestXHR.aspx",              //请求地址
        type: "POST",                       //请求方式
        data: { name: "super", age: 20 },        //请求参数
        dataType: "json",
        success: function (response, xml) {
            // 此处放成功后执行的代码
        },
        fail: function (status) {
            // 此处放失败后执行的代码
        }
    });

    function ajax(options) {
        options = options || {};
        options.type = (options.type || "GET").toUpperCase();
        options.dataType = options.dataType || "json";
        var params = formatParams(options.data);

        //创建 - 非IE6 - 第一步
        if (window.XMLHttpRequest) {
            var xhr = new XMLHttpRequest();
        } else { //IE6及其以下版本浏览器
            var xhr = new ActiveXObject('Microsoft.XMLHTTP');
        }

        //接收 - 第三步
        xhr.onreadystatechange = function () {
            if (xhr.readyState == 4) {
                var status = xhr.status;
                if (status >= 200 && status < 300) {
                    options.success && options.success(xhr.responseText, xhr.responseXML);
                } else {
                    options.fail && options.fail(status);
                }
            }
        }

        //连接 和 发送 - 第二步
        if (options.type == "GET") {
            xhr.open("GET", options.url + "?" + params, true);
            xhr.send(null);
        } else if (options.type == "POST") {
            xhr.open("POST", options.url, true);
            //设置表单提交时的内容类型
            xhr.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
            xhr.send(params);
        }
    }
    //格式化参数
    function formatParams(data) {
        var arr = [];
        for (var name in data) {
            arr.push(encodeURIComponent(name) + "=" + encodeURIComponent(data[name]));
        }
        arr.push(("v=" + Math.random()).replace("."));
        return arr.join("&");
    }
	
```


### 1、创建

1.1、IE7及其以上版本中支持原生的 XHR 对象，因此可以直接用： var oAjax = new XMLHttpRequest();

1.2、IE6及其之前的版本中，XHR对象是通过MSXML库中的一个ActiveX对象实现的。有的书中细化了IE中此类对象的三种不同版本，即MSXML2.XMLHttp、MSXML2.XMLHttp.3.0 和 MSXML2.XMLHttp.6.0；个人感觉太麻烦，可以直接使用下面的语句创建： var oAjax=new ActiveXObject(’Microsoft.XMLHTTP’);

### 2、连接和发送

2.1、open()函数的三个参数：请求方式、请求地址、是否异步请求(同步请求的情况极少，至今还没用到过)；

2.2、GET 请求方式是通过URL参数将数据提交到服务器的，POST则是通过将数据作为 send 的参数提交到服务器；

2.3、POST 请求中，在发送数据之前，要设置表单提交的内容类型；

2.4、提交到服务器的参数必须经过 encodeURIComponent() 方法进行编码，实际上在参数列表”key=value”的形式中，key 和 value 都需要进行编码，因为会包含特殊字符。每次请求的时候都会在参数列表中拼入一个 “v=xx” 的字符串，这样是为了拒绝缓存，每次都直接请求到服务器上。

    encodeURI() ：用于整个 URI 的编码，不会对本身属于 URI 的特殊字符进行编码，如冒号、正斜杠、问号和井号；其对应的解码函数 decodeURI()；
    encodeURIComponent() ：用于对 URI 中的某一部分进行编码，会对它发现的任何非标准字符进行编码；其对应的解码函数 decodeURIComponent()；


### 3、接收

3.1、接收到响应后，响应的数据会自动填充XHR对象，相关属性如下
    responseText：响应返回的主体内容，为字符串类型；
    responseXML：如果响应的内容类型是 "text/xml" 或 "application/xml"，这个属性中将保存着相应的xml 数据，是 XML 对应的 document 类型；
    status：响应的HTTP状态码；
    statusText：HTTP状态的说明；

3.2、XHR对象的readyState属性表示请求/响应过程的当前活动阶段，这个属性的值如下
  0-未初始化，尚未调用open()方法；
  1-启动，调用了open()方法，未调用send()方法；
  2-发送，已经调用了send()方法，未接收到响应；
  3-接收，已经接收到部分响应数据；
  4-完成，已经接收到全部响应数据；

只要 readyState 的值变化，就会调用 readystatechange 事件，(其实为了逻辑上通顺，可以把readystatechange放到send之后，因为send时请求服务器，会进行网络通信，需要时间，在send之后指定readystatechange事件处理程序也是可以的，我一般都是这样用，但为了规范和跨浏览器兼容性，还是在open之前进行指定吧)。

3.3、在readystatechange事件中，先判断响应是否接收完成，然后判断服务器是否成功处理请求，xhr.status 是状态码，状态码以2开头的都是成功，304表示从缓存中获取，上面的代码在每次请求的时候都加入了随机数，所以不会从缓存中取值，故该状态不需判断。

###　4、ajax请求是不能跨域的！

## 二、JSONP跨域

JSONP(JSON with Padding) 是一种跨域请求方式。主要原理是利用了script 标签可以跨域请求的特点，由其 src 属性发送请求到服务器，服务器返回 js 代码，网页端接受响应，然后就直接执行了，这和通过 script 标签引用外部文件的原理是一样的。

　　JSONP由两部分组成：回调函数和数据，回调函数一般是由网页端控制，作为参数发往服务器端，服务器端把该函数和数据拼成字符串返回。

　　比如网页端创建一个 script 标签，并给其 src 赋值为 http://www.superfiresun.com/json/?callback=process， 此时网页端就发起一个请求。服务端将要返回的数据拼好最为函数的参数传入，服务端返回的数据格式类似”process({‘name’:’superfiresun’})”，网页端接收到了响应值，因为请求者是 script，所以相当于直接调用 process 方法，并且传入了一个参数。

　　单看响应返回的数据，JSONP 比 ajax 方式就多了一个回调函数。

``` javascript
	function jsonp(options) {
        options = options || {};
        if (!options.url || !options.callback) {
            throw new Error("参数不合法");
        }

        //创建 script 标签并加入到页面中
        var callbackName = ('jsonp_' + Math.random()).replace(".", "");
        var oHead = document.getElementsByTagName('head')[0];
        options.data[options.callback] = callbackName;
        var params = formatParams(options.data);
        var oS = document.createElement('script');
        oHead.appendChild(oS);

        //创建jsonp回调函数
        window[callbackName] = function (json) {
            oHead.removeChild(oS);
            clearTimeout(oS.timer);
            window[callbackName] = null;
            options.success && options.success(json);
        };

        //发送请求
        oS.src = options.url + '?' + params;

        //超时处理
        if (options.time) {
            oS.timer = setTimeout(function () {
                window[callbackName] = null;
                oHead.removeChild(oS);
                options.fail && options.fail({ message: "超时" });
            }, time);
        }
    };

    //格式化参数
    function formatParams(data) {
        var arr = [];
        for (var name in data) {
            arr.push(encodeURIComponent(name) + '=' + encodeURIComponent(data[i]));
        }
        return arr.join('&');
    }
```

1、因为 script 标签的 src 属性只在第一次设置的时候起作用，导致 script 标签没法重用，所以每次完成操作之后要移除；

2、JSONP这种请求方式中，参数依旧需要编码；

3、如果不设置超时，就无法得知此次请求是成功还是失败；

4、调用方法

```javascript
	jsonp({
	    url:"http://www.baidu.com",
	    callback:"callback",   //跟后台协商的接收回调名
	    data:{id:"1000120"},
	    success:function(json){
	        alert("jsonp_ok");
	    },
	    fail:function(){
	        alert("fail");
	    },
	    time:10000
	})
```


## 三、模拟JQuery中的ready()事件

1、DOMContentLoaded事件，在DOM树加载完成之后立即执行，始终会在load之前执行。IE9+、FF、Chrome、Safari3.1+和Opera9+都支持该事件。

  对于不支持该事件的浏览器，可以使用如下代码：

``` javascript
	setTimeout(function(){
　　// 代码块
}, 0) ;
```

DOMContentLoaded 事件只能通过 DOM2 级方式添加，即采用addEventListener()/attachEvent() 方式添加才能够使用。事件对象不会提供任何额外信息。

2、readystatechange事件

 IE为DOM文档中的某些部分(区别于 XHR 对象的 readystatechange 事件)提供了该事件，这个事件的目的是提供与文档或元素的加载状态有关的信息，但这个事件的行为有时候也很难预料。支持该事件的对象都有一个readyState属性，注意，不是 event 事件对象。IE、Firefox4+和Opera 支持该事件。

readyState属性的值如下：
    “uninitialized” - 未初始化：对象存在但尚未初始化；
    “loading” - 正在加载：对象正在加载数据；
    “loaded” - 加载完毕，对象加载数据完毕；
    “interactive” - 交互：可以操作对象了，但还没有完全加载；
    “complete” - 完成：对象已经加载完成；

2.1、并非所有的对象都会经历readyState的这几个阶段，如果这个阶段不适用某个对象，则该对象完全可能跳过该阶段，并没有规定哪个阶段适用于哪个对象。这意味着 readystatechange 事件经常会少于4次，相对应的 readyState 属性值也不是连续的。

2.2、对于 document 而言，interactive 和 complete 阶段会在于 DOMContentLoaded 大致相同的时刻触发 readystatechange 事件；

　　load 事件和 readystatechange 事件的触发顺序会因页面的外部资源的多少而变化，也就是说，readystatechange 事件并不会一直在 load 事件之前执行。外部资源越多，对 readystatechange 事件就越有利。

　　interactive 和 complete 的顺序也是不一致的，谁都有可能先执行，引用的外部资源越多，对交互阶段越有利。所以为了尽可能早的执行代码，两个状态要同时判断。

3、doScroll 
IE5.5+支持，当页面中有滚动条时，可以用 doScroll("right")/doScroll("down") 等来移动滚动条，这个方法只有等DOM加载完成以后才能用，所以在IE低版本浏览器中可以通过这个属性判断 DOM 结构是否加载完成。介绍这个属性主要是模仿 jquery 中的解决方案。

``` javascript
	function ready(readyFn) {
        //非IE浏览器
        if (document.addEventListener) {
            document.addEventListener('DOMContentLoaded', function () {
                readyFn && readyFn();
            }, false);
        } else {
            //方案1和2  哪个快用哪一个
            var bReady = false;
            //方案1
            document.attachEvent('onreadystatechange', function () {
                if (bReady) {
                    return;
                }
                if (document.readyState == 'complete' || document.readyState == "interactive") {
                    bReady = true;
                    readyFn && readyFn();
                };
            });

            //方案2
            //jquery也会担心doScroll会在iframe内失效，此处是判断当前页是否被放在了iframe里
            if (!window.frameElement) {
                setTimeout(checkDoScroll, 1);
            }
            function checkDoScroll() {
                try {
                    document.documentElement.doScroll("left");
                    if (bReady) {
                        return;
                    }
                    bReady = true;
                    readyFn && readyFn();
                }
                catch (e) {
                    // 不断检查 doScroll 是否可用 - DOM结构是否加载完成
                    setTimeout(checkDoScroll, 1);
                }
            };
        }
    };
```

注：
setTimeout(checkDoScroll, 1); 目的是让浏览器尽快执行 checkDoScroll 函数，间隔时间设置为 1ms，对当下的浏览器来说是不太可能的。每个浏览器都有自己默认的最小间隔时间，即使时间设置为最小间隔时间，也只是代表隔这些时间过去之后，JavaScript 会把 checkDoScroll 加入到执行队列中，如果此时 JavaScript 进程空闲，则会立即执行该代码。
