---
title: deepin 上多版本 php 版本的配置
tags:

 - deepin
 - 环境
date: 2019-09-18
---


## 1@ 前言

之前博客上的一篇文章，稍有改动。

## 2@ 过程

### 2.1 什么是ppa

首先需要了解下**ppa**：

大多数linux的软件管理包，如 centos 的 yum，debain 的 apt 一般都是不断更新自己软件库里面的一些应用，所以我们默认下载到的应当是最新版本的软件，那么如果软件仓库里面的软件被撤下，我们又想下载老版本的时候，就会用到**ppa**这个东西

**ppa**（personal package archive），即个人包档案，里面可以有软件制作者发布的各个版本，一般来说是用来寻找最新版的软件，以防止上述的软件管理包未更新，此处我们用来下载旧包。

由于deepin也是debain系列的系统，所以可以使用ubuntu的添加源的办法来添加ppa源。

首先将源加进到本机的源文件中，然后导入公钥，更新源（如果有依赖需要下一下dirmngr）：

```
echo "deb http://ppa.launchpad.net/ondrej/php/ubuntu xenial main" | tee -a /etc/apt/sources.list

echo "deb-src http://ppa.launchpad.net/ondrej/php/ubuntu xenial main" | tee -a /etc/apt/sources.list

apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E5267A6C
```

添加源如果有问题，加上这句：

```
sudo apt-get install unknown-horizons
```

### 2.2 下载php多版本

网上发现了一个不错的文章，里面刚好讲到ubuntu中多版本php的安装，deepin上安装过程大同小异。

```
sudo apt-get install -y php5.6-common php5.6-mbstring php5.6-mcrypt php5.6-mysql php5.6-xml php5.6-gd php5.6-curl php5.6-json php5.6-fpm php5.6-zip php5.6-mcrypt libapache2-mod-php5.6

sudo apt-get install -y php7.0-common php7.0-mbstring php7.0-mcrypt php7.0-mysql php7.0-xml php7.0-gd php7.0-curl php7.0-json php7.0-fpm php7.0-zip php7.0-mcrypt libapache2-mod-php7.0

sudo apt-get install -y php7.1-common php7.1-mbstring php7.1-mcrypt php7.1-mysql php7.1-xml php7.1-gd php7.1-curl php7.1-json php7.1-fpm php7.1-zip php7.1-mcrypt libapache2-mod-php7.1
```

上述安装了三个版本，分别是**5.6 7.0 7.1**

下面使用**a2enmod a2dismod**命令来开启或禁用系统的某个模块来完成php版本的快速切换。

```
sudo a2enmod rewrite #开启重写转向
sudo a2enmod headers
```

然后在你的shell配置文件 **~./bashrc 或者 ~/.zshrc** 中添加别名来达到快速切换的目的，原理就是禁用其他php模块，激活要使用的php版本

```
alias php56='sudo a2dismod php7.0 && sudo a2dismod php7.1 && sudo a2enmod php5.6 && sudo service apache2 restart'
alias php70='sudo a2dismod php5.6 && sudo a2dismod php7.1 && sudo a2enmod php7.0 && sudo service apache2 restart'
alias php71='sudo a2dismod php5.6 && sudo a2dismod php7.0 && sudo a2enmod php7.1 && sudo service apache2 restart'
```

然后 `source ~/.zshrc 或者 /.bashrc`

![img](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/DEEPIN/%E6%B7%B1%E5%BA%A6%E6%88%AA%E5%9B%BE_%E9%80%89%E6%8B%A9%E5%8C%BA%E5%9F%9F_20190522162731.png)

检查版本，默认只显示最高版本，使用上述别名会成功切换php的版本，只不过不会显示在此处。

### 2.3 vscode + xdebug

下个vscode就感觉它的调试十分简便，下面我们来配置下xdebug的断点调试，这里与普通安装php可能有些不同，需注意。

首先，到 xdebug 官网查找对比对应的php版本下的xdebug版本。使用刚刚的别名，切换下不同状态下的php版本，写个 phpinfo() 的脚本来输出到 `https://xdebug.org/wizard.php`,点击 analyse 来分析下应当下载的xdebug版本，这里我直接贴上我的测试结果

```
5.6 --->20131226
7.0 --->20151012
7.1 --->20160303
```

其实这个强大的网站会自动定位 phpinfo 信息中的配置文件的位置，然后有具体的操作步骤,而生成不同版本对应的 xdebug，下载安装包即可，可以利用 phpize 这个工具来生成对应版本的 xdebug 文件，如果没有可以下载对应版本的扩展：

`sudo apt-get install php5-dev`

**建议扩展从低版本开始下载，因为高版本扩展会优先被识别。**接下来的操作分别是：

- ./configure
- make
- make install

然后将对应生成的 xdebug 文件配置信息放到对应 php.ini 文件中：

【这里以5.6 版本为例，其他版本的配置同理，位置 /etc/php/5.6/apache2/php.ini】

将下面这段配置加入到该配置文件中

```
zend_extension="/usr/lib/php/20131226/xdebug.so"      #之后配置其他版本php的时候只需将这个文件的位置换成上面我贴出的对应文件即可                                                                                                                      

xdebug.remote_enable = 1

xdebug.remote_handler = dbgp

xdebug.remote_mode = req 

xdebug.remote_host = 127.0.0.1

xdebug.remote_port = 9002

xdebug.remote_enable = 1

xdebug.remote_autostart = 1
```

然后保存，在对应版本的php-ini中末尾填上`[xdebug]` 来激活xdebug扩展。

最后重启，php56,来检查下是否配置成功。

![img](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/DEEPIN/%E6%B7%B1%E5%BA%A6%E6%88%AA%E5%9B%BE_%E9%80%89%E6%8B%A9%E5%8C%BA%E5%9F%9F_20190522164713.png)

配置成功！愉快的去审计叭～

## 3 @ 总结

推荐下黄老板和比伯的新歌 **<I Don’t care>**，已经被洗脑了。真好～

Reference：

`https://www.jianshu.com/p/dedd2818c7f9`

`https://zhidao.baidu.com/question/1670867120091087107.html`

`https://blog.csdn.net/zhousmq/article/details/77765451`

`https://www.cnblogs.com/wangshuyi/p/8512036.html`
