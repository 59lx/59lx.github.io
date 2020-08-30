---
title: Xss 攻击向量解析的探析
tags:

 - xss
 - javascript
 - tips
date: 2019-09-17
---



# 1@ 前言

当利用编码来绕过有关 xss 的 waf 的时候，有时我们构造的 payload 得不到执行。纵然你的编码方式是对的，但是如果对应的解析器不解析你的攻击向量，也是徒劳的。这时候我们就需要比较清晰的了解自己的 payload 所遵守的解析策略与经历的解析流程，然后构造出更加有针对性的攻击向量。



# 2@ 基础铺垫

在了解一个 payload 解析流程之前，我们需要先熟悉一些用到的基础知识。

## 2.1 HTML字符实体

在 HTML 中，字符实体其实就是转义序列，一般是为了正确显示保留，如 **‘<’**, **‘>’**, **空格**等。HTML 中的实体转义格式有以下两种：

### Entity name （实体名称）

格式 ： **&实体名;**

eg ： `&lt;` （less than）表示 ‘<’

[更多的字符实体名戳这里](http://www.w3school.com.cn/html/html_entities.asp)

### Entity Number (实体编号)

格式 ： **&实体编号;**

其中实体编号可以 是 16 进制格式 **&#x** 开头，或者 10 进制格式的 **&#** 开头。

eg ： `&#65;` 表示字符 ‘A’ 



## 2、 JS 中的 unicode 表示

格式 ：**\u + 16 进制的 unicode 编码**

eg : `\u5b89` 表示汉字 **安**



# 3@ 攻击向量经历的三种解析

## 1、HTML 解析

原版的 HTML 解析规则[戳这](http://www.whatwg.org/specs/web-apps/current-work/multipage/tokenization.html)。

eg : `<a href="http://www.baidu.com">text</a>`

联想一下我们学过的数电，可以视 HTML 解析器为一个状态机，遍历获得的字符，按照规则进行状态的转换。以上面的例子：

当碰到 **<** 字符的时候，就会转换到标签开始状态 (Tag open state)【前提是紧跟字符不为 **/**】

然后跳转到 标签名状态（Tag name state) 【对应例子里面的 a 】

. . . . . .

进入数据状态 (Data state) 【对应例子里面的 **text** 】, 在刚进入这个状态时，释放当前标签的 token 【对应例子里面的 a 标签的前部分】。在数据状态时，每发现一个完整的标签，就会释放一个 token.

那么，是否存在这样一个位置状态，它可以将我们的字符实体解析并赋予本来的非字符的特殊意义(如 `&lt;`表示 **<**可以当标签开始符来对待，然后让其中的脚本执行)? 答案是：没有。这就与 HTML 解析 规则有关。我们再拿一个例子来讲：

`<div>&lt;script>alert(1);</script></div>`

因为 div 标签开始后，解析器还是在数据状态，不会将这个字符引用转换为 ‘标签开始状态’，所以就可以保证我们得到的字符实体引用只会被解析为单纯的数据，而没有特殊意义，从而从一定程度上杜绝了一些风险。 ![xss_1](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/xss_payload_new/xss_1.png)

从图中我们可以看出来，’<’ 标签已经被转义，但是只是作为纯字符输出在页面中。之后的 由于未配对成功而被解析器去除。

而在 HTML 中也并不是所有位置都会解析转义字符引用，有三种情况下会转义：

### 数据状态的解析

`<div>&lt;&gt;</div>`

![xss_2](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/xss_payload_new/xss_2.png)

可以看到字符引用被解析。

### 属性值中的解析

`<a href="javascript:alert&#40;1)"></a>`

![xss_3](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/xss_payload_new/xss_3.png)

可以看到 ‘(‘ 被解码。

### RCDATA状态下的字符引用

这时我们需要先了解下 HTML 中的 5 类元素

```
1.  空元素(Void elements)，如<area>,<br>,<base>等等

2.  原始文本元素(Raw text elements)，有<script>和<style>

3.  RCDATA元素(RCDATA elements)，有<textarea>和<title>

4.  外部元素(Foreign elements)，例如MathML命名空间或者SVG命名空间的元素

5.  基本元素(Normal elements)，即除了以上4种元素以外的元素

五类元素的区别如下：

1.  空元素，不能容纳任何内容（因为它们没有闭合标签，没有内容能够放在开始标签和闭合标签中间）。

2.  原始文本元素，可以容纳文本。

3.  RCDATA元素，可以容纳文本和字符引用。

4.  外部元素，可以容纳文本、字符引用、CDATA段、其他元素和注释

5.  基本元素，可以容纳文本、字符引用、其他元素和注释
```

 

由于 RCDATA 状态中可以转义字符实体，那么在像 **textarea** **title** 等标签中就可以字符引用。这里 还有一个特殊的点就是在 **RCDATA** 元素标签 的内容中，遇到 `<` 后，会进入 **RCADATA小于号状态**，如果紧接着不是`/`字符的话，就会返回到 **RCDATA状态**。那么也就是说在 **textarea**或者**title**标签中只能认识并解析的标签只有对应的 **</textarea>**或者 **</title>**,不会有新标签的产生，也就杜绝了脚本的执行。例如：

`<textarea><script>alert(6)</script></textarea>`

是不会执行的。

## 2、URL 解析

同样的，URL 解析器依旧可以看做一个状态机，其解析规则在[这里](http://url.spec.whatwg.org/)。这里只提几个 xss_payload 能涉及到的地方。

1、首先，资源类型必须是大写字母或者小写字母，否则会进入到无状态类型。那首先就要求资源的协议要保证格式完全的正确,譬如使用 **javascript:** 伪协议时不能使用编码来绕过，因为这会让 URL 解析器跳转到 ‘ 无类型 ’状态，导致资源不能正确解析，我们构造的 payload 也就不能得到正确的执行，我们来看看下面的例子：

`<a href="%6a%61%76%61%73%63%72%69%70%74:alert(1)">text</a>` 编码了 **javascript**，可以看到无法正确执行。

![xss_4](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/xss_payload_new/xss_4.png)

从图中可以看出，得到的跳转地址是 **http://localhost/javascript:alert(1)**，原因就是没有正确的识别资源的协议类型，拼接到了默认地址的后面，导致 payload 无法执行。

2、其次，URL 编码过程使用的是 UTF-8 编码，如果给 URL 进行别种方式的编码，会导致 URL 解析器不能正确识别。

### HTML 中的两种 URL资源的执行

我们可以利用伪协议来进行 xss 的反射攻击，而在HTML 标签中有两种 url 的连接类型，一种是 `src`，一种是 `href`，我们先来看看两者的区别

#### 1、src

src 适用于替换当前元素，是 source 的缩写，指向外部资源的位置，指向的内容会 **自动的** 嵌入到文档中标签的位置。请求 src 资源的时候会将其指向的资源下载并应用到文档内。例如 js 脚本，img 图片，frame 元素等。

#### 2、href

href 是 Hypertext Reference 的缩写，指向网络资源的所在位置，是与该页面有关联的，是引用。

**src**与 **href** 的主要区别就是 一个是引入，一个是引用。

之前在很多地方看到 payload 都有这种 src 引用的格式

`<img src="javascript:alert(0);">`

但是现在在 Chrome 与 Firefox 上实验了下都不能执行。原因就是当下的一部分浏览器**为了防止恶意代码的注入，已经禁用了 src 属性的 javascript 的伪协议**。我们还有一种引入外部 js 文件的 src 利用方式：

`<script src="evil.js"></script>`

其中 evil.js 中的内容为

```
alert(/xss/);
```

## 3、 Javascript 解析

详见 javascript 的[解析规则](http://www.ecma-international.org/publications/standards/Ecma-262.htm)

因为 js 脚本一般是内嵌在 HTML 页面中的，所以 javascript 的解析容易与 HTML 解析搞混了。Javascript 的解析需要注意的点有下面两点：

### script 标签的解析

在之前，我们提到过 HTML 中的 5 类元素，其中 **script** 标签属于 **原始文本数据**,。它有个特别的解析属性就是虽然内嵌在了 HTML 界面的数据状态，但是仍然不能解析字符引用。看看下面这个例子：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
    <script>&lt;</script>
    &lt;
</body>
</html>
```

![xss_5](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/xss_payload_new/xss_5.png)

可以看到，里面的字符引用并没有被解码。

### unicode 字符编码的解析

在之前的基础铺垫的时候提到了 js 中 unicode 的编码问题，那么它们在脚本的什么位置会被解析？又会被解析成什么意思？unicode 大致有以下三个位置会被解析，但是解析的意义有些许不同。

#### 1、字符串中

unicode 放在字符串中的时候会被解析，但是只是会解析为常规的字符，不会解释为破坏上下文的特殊意义的字符。请看下例子：

```html
<script>
      var str = '\u0027';
      document.write(str);
</script>
```

![xss_6](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/xss_payload_new/xss_6.png)

可以看到，被编码的单引号只是解释为了单纯的字符，没有破坏 js 的上下文。

#### 2、标识符中

标识符包括函数名，属性名等。如果使用 unicode 编码这些数据的话，会被正确解释为上下文中的标识符，可以正确的运行。看下面这个例子：

```javascript
<script>\u0064\u006f\u0063\u0075\u006d\u0065\u006e\u0074.\u0077\u0072\u0069\u0074\u0065('test_successfully@');</script>
```

注意这里只编码了 **document** 和 **write**，并没有编码点号，因为测试结果显示无法编码点号进行执行，但这并不影响我们探测，因为一般利用到的函数都等都只是单纯的字母拼接。

![xss_7](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/xss_payload_new/xss_7.png)

成功解码并执行了打印函数。

#### ３、控制字符

当控制字符不在引号包围的时候，例如 `(` `)` `'` `"`这些字符，裸露在文档外面进行 unicode 编码，js 解析器不能将它们正确的解析为控制字符，仅仅会被解码为标识符亦或是字符串，所以，想要成功执行一个 js 函数，就必须保证括号为 **()**，而不是 **\u0028 \u0029**.

综上，js 在解析 unicode 编码的时候，能赋予非字符意义的地方有且仅有**标识符**这一个位置。

# 4@ 解析流的顺序

前端这三大解析器一般是相互配合的，我们需要搞清楚他们配合的顺序，然后对应构造 payload ，才有可能正确的被解码执行。

一般的解析顺序如下：

**1、HTML 解析**

浏览器从网络堆栈中获取内容后，会触发 HTML 解析器根据解析规则（上文链接）对文章进行词法的解析，在这一步中，所有的**合理的字符引用都会被解码**【合理即能够被解析的位置放入字符引用】，解析完成后 DOM 树就被创建好了。

**2、JS or URL 解析**

为什么第二步的解析顺序不确定呢，这与文档资源类型的顺序有关。其中 Js 解析器 负责文档中内嵌脚本，unicode 编码等的解析，URL 解析器负责解析碰到的 URL 资源的解码。但位置不同，可能会造成不同的解析顺序。试看下两例：

```javascript
脚本1：
 <a href="javascript:%61%6c%65%72%74%28%31%29">sequence_test</a> 编码了 alert(1)
 
脚本2：
<a href=# onclick="window.\u006f\u0070\u0065\u006e('http://www.baidu.com')">sequence_test</a>   
    编码了open
```

其中

脚本1先是由 url 解析器解析 href 中的资源，解析完成后进入 js 的上下文，所以 js 解析器再解析 js 的内容。

脚本2先是由 js 解析器解析 onclick 中的 unicode 编码，编码完成后发现参数是 url 的上下文，这时 url 解析器介入解析。

可以看到，不同的位置会导致不同的解析顺序。

# 5@ xss_payload 的练习

下面给出了常见的几种弹窗的方法，通过上面的阅读与学习，来判断下是否可以成功执行代码。

```javascript
1、  <a href="%6a%61%76%61%73%63%72%69%70%74:%61%6c%65%72%74%28%31%29"></a>    
    url编码了 javascript:alert(1)
 
2、  <a href="&#x6a;&#x61;&#x76;&#x61;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;:%61%6c%65%72%74%28%32%29">xss_test</a>   
	字符实体编码了 javascript url 编码了 alert(2)    

3、  <a href="javascript%3aalert(3)">test</a>
    url 编码了 ：

4、  <div>&#60;img src=x onerror=alert(4)&#62;</div>  
    字符实体编码了尖括号

5、  <textarea>&#60;script&#62;alert(5)&#60;/script&#62;</textarea> 
	字符实体编码了尖括号
```

### 解析

1、 无法执行，因为编码了协议

2、 可以执行，html 先解析字符实体编码 javascript ，url 解析器再解析 alert(2)，位置与格式都允许。

3、 无法执行，因为协议被编码导致无法正确识别协议。

4、 无法执行，因为嵌在数据状态内，只能解析字符引用为普通字符串，无特殊意义。

5、无法执行，首先是因为在数据状态中只能将尖括号转义为字符串，其次 RCDATA 元素中不能嵌入新的标签，嵌入也无法执行代码。

更多例子附带答案，[戳这](http://test.attacker-domain.com/browserparsing/answers.txt)

# 6@ 关于 svg 标签中解析 js 代码

上面写道 **svg** 标签属于外部元素，**script** 标签属于原是文本元素，而且也知道 **script** 标签里面是不能转义字符实体编码的，那么我们考虑一下下面这段代码是否可以弹窗：

`<svg><script>alert&#40;/xss/)</script></svg>`

按我们上面理解的话，`<script>` 标签下不能解析字符引用，就算可以解析，圆括号这种被放在数据状态下的解码也只能成为字符串，并不能被解析为控制字符。

所以不能码？上图～

![xss_8](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/xss_payload_new/xss_8.png)

![xss_9](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/xss_payload_new/xss_9.png)

太夸张了，不仅解码了字符引用，还转换为了控制字符并且弹窗了～

按照上面的分析，问题绝对就在 **svg** 这个标签上面。

`<svg>`这个标签 使用 XML 格式定义图形，XML 会解析字符实体，而 w3c 上面给出的这段实例代码中也可以看出 xml 使用的端倪。

```html
<!DOCTYPE html>
<html>
<body>

<svg xmlns="http://www.w3.org/2000/svg" version="1.1" height="190">
  <polygon points="100,10 40,180 190,60 10,60 160,180"
  style="fill:lime;stroke:purple;stroke-width:5;fill-rule:evenodd;" />
</svg>

</body>
</html>
```

那么为什么 **script** 标签可以被解析呢？原因就是 **svg** 标签支持 嵌入 **script** 标签。

这是支持 **script** 标签的[文档](https://www.w3.org/TR/SVG11/script.html)借用引用文章作者的话来讲就是：

这个 payload 之所以可以执行是因为遵循了 **svg** 和 **xml** 的标准。

# 7@ 总结

理解一个漏洞的利用，需要也有必要挖的深些，这样才能更好的从攻与防的角度来构造攻击向量，亦或是根据攻击的思路来构造防御的 waf。

`Reference`:

`https://www.cnblogs.com/polk6/p/html-entity.html`

`https://www.cnblogs.com/liuxianan/p/display-unicode-character-in-html-css-and-js.html`

`https://zhidao.baidu.com/question/583316301.html`

`https://bbs.csdn.net/topics/392055451`

`http://test.attacker-domain.com/browserparsing/answers.txt`

`http://test.attacker-domain.com/browserparsing/tests.html`

`http://bobao.360.cn/learning/detail/292.html`

`https://www.hackersb.cn/hacker/85.html`

`http://www.w3school.com.cn/html5/html_5_svg.asp`

`https://www.w3.org/TR/SVG11/script.html`
