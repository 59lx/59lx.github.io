---
title: ntfs 文件系统中的供选数据流
tags:
 - ntfs
 - 方法
date: 2019-08-17
---


## 1@ 前言

前几天在审计 seacms 的时候碰到了一个问题十分有趣。上图说话：

 ![1.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/ntfs%E7%B3%BB%E7%BB%9F/1.png)



 ![2.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/ntfs%E7%B3%BB%E7%BB%9F/2.png)

上两图反映的问题就是在 windows 系统中，一个文件名中存在 `:` 的文件，在文件夹是看不到冒号后面完整的文件名的，但是这个文件的的确确存在，而且文件内容也可以成功的写入，亦或是读取。

在思索无果后，上小密圈提问得到了 wonderkun 师傅的指点，在 ll 的指引下，学习了一下ntfs 下的供选数据流，在此做笔记，也供有相同疑惑的小伙伴们参阅。

## 2@ ADS 供选数据流

### 2.1 MFT 主文件表

ntfs 文件系统使用 Master File Table（MFT）即主文件表来管理文件。下面是百度百科对于 MFT 的简略介绍：



> MFT，即主文件表（Master File Table）的简称，它是NTFS文件系统的核心。MFT由一个个MFT项（也称为文件记录）组成，每个MFT项占用1024字节的空间。每个MFT项的前部几十个字节有着固定的头结构，用来描述本MFT项的相关信息。后面的字节存放着“属性”。每个文件和目录的信息都包含在MFT中，每个文件和目录至少有一个MFT项。除了引导扇区外，访问其他任何一个文件前都需要先访问MFT，在MFT中找到该文件的MFT项，根据MFT项中记录的信息找到文件内容并对其进行访问。NTFS(New Technology File System)，是一种新型文件系统。



windows 的每个文件都对应一个 MFT 记录，每个记录由诸多属性组成。譬如我们现在有一个叫做 test.txt 的文件，内容为 **Hello,world!** , 那么它的 MFT 结构如下图：

 ![3.jpg](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/ntfs%E7%B3%BB%E7%BB%9F/3.jpg)



我们通过这个实例来学习下 MFT 的属性。

- $FILE_NAME 属性包含文件名，创建，修改，访问的时间。这里的文件名为 test.txt
- $DATA 包含文件具体内容。这要注意的是 ，此例文本内容偏小，小于1kb，所以直接存储在 MFT 这个结构里面了，称作 Resident（理解为本地居民）。如果文件内容大于1kb，该位置就会放置改文本内容的具体存储地址，类型也会变为 non-resident。

### 2.2 MFT 的多文件属性

通过上面我们了解了 MFT 的大致结构，而它并不仅仅是具有一个 $DATA 属性的。一个文件可以有多个 $DATA 属性。这里为了方便查看，我们还是来看看引用文章上面的例子。当下我们向 test.txt 加入一个 ThisIsAnADS 的 $DATA属性:

`echo Hello,freebuf! > test.txt:ThisIsAnADS`

其 MFT 结构就变成了下面这样：

 ![4.jpg](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/ntfs%E7%B3%BB%E7%BB%9F/4.jpg)



这里可以看到多出来一个 $DATA 属性，而且还有名字，就是我们起的 ThisIsAnADS 。我们平常存储在文件里的都是称为主数据流(primary data stream)，默认没有名字。而之后创建的数据流就叫做供选数据流(alternate data stream),即 ADS，原文章作者说其翻译为**供选数据流**更合适，我也有同感。翻译为交换数据流并没有体现出交换两个字的过程。

通常意义下，ADS 对用户来说都是表层可视化隐藏的，譬如前言例子中的文件，打开是无内容的。

 ![5.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/ntfs%E7%B3%BB%E7%BB%9F/5.png)



单单看大小确实如此。我们先略过占用空间不为0的这个现象。后面会讨论到。

下面，我们先来看看如何看到隐藏在供选数据流里面的数据。

### 2.3 供选数据流的使用

由于隐蔽性较强，这一技术被广泛的应用到了木马，后门文件的隐藏上面。我们不仅可以隐藏文本数据到 ads 中，还可以是图片数据，亦或是可执行的 exe 文件数据。方法相似：

#### 1# 寄生在文件上

同上例，我们可以将图片隐藏进入另一个文件的供选数据流里面,此时使用到 type 命令，作用相似于 linux 下的cat 命令，具体用法[戳这](https://blog.csdn.net/grey_csdn/article/details/69922844)。

```shell
type shell.png >> 1.txt:test.png
```

我们将图片数据寄生在文件 1.txt 上面

#### 2# 寄生在文件夹上

```shell
type shell.png >> test:test.png
```

将图片数据寄生在文件夹 test 上。

### 2.4 查看供选数据流

现在我们讲讲如何查看 ads 数据。

#### 1# 系统自带应用查看

- 文本文件的话就用记事本 【notepad + 文件名】打开。

- 图片文件就是用画板 【mspaint + 文件名】打开。
- 可执行文件就用 【start + 文件名】 打开。

这里需要注意的是，可能考虑到安全因素，微软在 windows_Xp 之后就禁止直接执行供选数据流中的可执行文件了，但是是存在绕过方法的，有兴趣的同学可以[戳这](https://www.freebuf.com/articles/73270.html)进去看看。

我们这里沿用上面的例子，打开寄生在文件夹下面的图片

`mspaint test:test.png`

 ![6.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/ntfs%E7%B3%BB%E7%BB%9F/6.png)



#### 2# cmd 下使用 dir /r 命令查看

还是以上面图片为例，在文件存在的路径下面打开 cmd：

`dir /r test:test.png`

 ![7.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/ntfs%E7%B3%BB%E7%BB%9F/7.png)



由于之前我测试的时候还添加了几次，所以查看 ads 数据流的时候还有其他文件。这也从侧面反映了一个问题，就是可以在一个文件或者文件下添加多个供选数据流，而且它们互不影响。



#### 3# powershell 下的 Get-Content 指令查看

指令格式

`Get-Content [被寄生的文件/文件夹] -stream [ads数据流名]`

```powershell
PS C:\Users\rt95\Desktop\test> Get-content test -stream test.png     
```

 ![8.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/ntfs%E7%B3%BB%E7%BB%9F/8.png)

可以看到，这条指令是直接可以读出数据内容的，用来读取寄生的 ads 文本数据会非常方便。

## 3@ 几个问题

### 3.1 供选数据流的大小

刚刚我们看到，存储的文件大小和占用空间不一致。证明供选数据流是会占用磁盘空间的。侧面反映出 windows 系统查看文件属性的时候，显示文件大小只是主数据流的大小，而占用空间则会较为真实的反映主数据和供选数据流的总大小。

### 3.2  linux 下对 ads 的读取

有时会出现在 linux 系统下读取 windows 文件的情况，这时我们需要挂载磁盘的读取方式，或者使用 ntfs-3g 程序进行 ntfs 文件系统的文件读写。

> NTFS-3G 是一个由 [Tuxera](http://zh.wikipedia.org/wiki/Tuxera) 公司开发并维护的开源项目。目的是为 Linux 提供 NTFS 分区的的驱动程序。可以安全高速的对 Windows NT （包含 Windows 2000、Windows XP、Windows Server 2003 和 Windows Vista）的文件系统进行读写。

可以使用 apt 或者 yum 下载 ntfs-3g。

然后我们创建一个 含有 ads 的文本文件。

```shell
touch 1.txt
echo hello,rt95 > 1.txt:2.txt
cat 1.txt:2.txt
```



可以看到文件内容被显示。

 ![9.png](https://raw.githubusercontent.com/59lx/userful_photo/master/myblog_photos/ntfs%E7%B3%BB%E7%BB%9F/9.png)





## 4@ 总结 

实习结束了，学到了不少东西，有感知上的，有知识上的。希望剩下的大学时光能不辜负自己的一腔热血吧~

加油 :)





Reference:

`https://baike.baidu.com/item/MFT/1185355?fr=aladdin`

`https://www.freebuf.com/articles/73270.html`

`http://www.huike007.cn/?p=150`

`https://blog.csdn.net/alone_map/article/details/51851071`

`https://www.qingsword.com/qing/812.html`
