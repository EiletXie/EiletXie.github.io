---
title: Tomcat的日志切分与定时删除
date: 2019-10-15 16:30:22
tags:
- tomcat
categories:
- JAVA
toc: true


---



在我负责的一个小系统中，Linux环境下，由于默认日志都是写入在 cattalina.out中，

我查看日志catalina.out 竟然已经到了 40G了，我想做一下 文件内容检索来追踪问题都无法进行。

于是我决定删除以前的无用日志，以每日作为单位将其进行切分，并写一个定时脚本，定时删除1个月以前的日志数据。

<!--more-->

Google了一下，大部分都是使用 Cronolog 工具进行切分日志，这里我也是第一次接触，记录一下操作过程方便日后再次使用。



话不多说，先看 cronolog 的官网介绍和下载

这里我试着用 yum install cronolog 发现不行，阿里云镜像找不到，就下载包的方式进行安装了。

github上有其项目，网址 https://github.com/fordmason/cronolog

<br/>

### cronolog简易介绍：

*CRONOLOG version 1.6.1  "cronolog" is a simple program that reads log messages from its input and writes them to a set of output files, the names of which are constructed using a template and the current date and time.  The template uses the same format specifiers as the Unix date command (which are the same as the standard C strftime library function).  "cronolog" is intended to be used in conjunction with a Web server, such as Apache, to split the access log into daily or monthly logs.* 

<br/>

**大体意思：**

​      “ cronolog”是一个简单程序，可从其输入中读取日志消息并将它们写入一组输出文件，其名称为使用模板名和当前日期和时间进行构造。 

模板使用与Unix date命令相同的格式说明符（与标准C strftime库函数相同）。

​      “ cronolog”旨在与Web服务器结合使用，例如 作为Apache，将访问日志分为每日或每月日志。



文件下载 github上的是 zip包，适合windows的部署，

至于 cronolog的详细使用方法和参数介绍，可以 github上的 readme.txt，介绍的还行，我这里就大致了解一下，后续有需要再回来增加用法。

Linux的tar.gz 包下载地址： https://fossies.org/linux/www/old/cronolog-1.6.2.tar.gz/cloc.html

{% asset_img cronolog.jpg cronolog%}

<br/>

### cronolog的安装

```shell
2、解压缩

> tar -zxvf cronolog-1.6.2.tar.gz

3、进入cronolog安装文件所在目录

> cd cronolog-1.6.2

4、运行安装 [进入文件夹，使用./configure命令进行编译，可以加--prefix指定安装目录]

> ./configure


> make

> make install

5、查看cronolog安装后所在目录（验证安装是否成功）

 > which cronolog

一般情况下显示为：/usr/local/sbin/cronolog


```

<br/>

### catalina.sh的修改

进行 TOMCAT目录下的 bin文件夹下

修改 catalina.sh 文件

注意： 这里假设你是TOMCAT7及版本以上，不过因为我有一个TOMCAT6的项目要维护，下面给出了TOMCAT6切分日志的步骤

需要修改3处地方：

1、 将

```shell
 if [ -z "$CATALINA_OUT" ] ; then CATALINA_OUT="$CATALINA_BASE"/logs/catalina.out fi 
```

改为

```shell
if [ -z "$CATALINA_OUT" ] ; then   CATALINA_OUT="$CATALINA_BASE"/logs/catalina.%Y-%m-%d.out   fi 
```



2、将

```shell
 touch "$CATALINA_OUT" 
```

注释掉

```shell
#touch "$CATALINA_OUT" 
```



3、 将

```shell
org.apache.catalina.startup.Bootstrap "$@" start \ >> "$CATALINA_OUT" 2>&1 "&" 
```

改为

```shell
org.apache.catalina.startup.Bootstrap "$@" start 2>&1 \ | /usr/local/sbin/cronolog "$CATALINA_OUT" >> /dev/null & 
```

<br/>

#### 注意：

1、有两处都改了，以前的内容不要注释，要直接删除了，不然会引发log报错

2、以上的“/usr/local/sbin/cronolog”配置的是cronolog的安装目录，这里要根据你的cronolog安装目录进行配置。

配置完成之后，重启tomcat就可以了。重启访问应用之后就会发现，Catalina.out不会再输出日志，日志会输入到一个catalina.日期.out的文件中。

{% asset_img catalina.jpg catalina%}

<br/>

**修改完该3处内容后重启tomcat**



**如果是TOMCAT6,没有 CATALINA_OUT变量名的话，第一步跳过，第二步要，第3步改成这个，其实就是因为  CATALINA_OUT="$CATALINA_BASE"/logs/catalina.%Y-%m-%d.out 进行替换即可。**

<br/>

```shell
org.apache.catalina.startup.Bootstrap "$@" start 2>&1 \  | /usr/local/sbin/cronolog "$CATALINA_BASE"/logs/catalina.%Y-%m-%d.out >> /dev/null & 
```

亲测这种写法在tomcat6可行!

{% asset_img show1.jpg show1%}

<br/>

### 效果示意：

未加切分日志时的catalina文件

{% asset_img show2.jpg show2%}



加了 Cornolog 工具进行切分后的log文件夹显示

{% asset_img show3.jpg show3%}



新打印的日志已按照每日来划分

因为这句话，如果数据量较小，你也可以按照每月为单位进行切分  catalina.%Y-%m.out

```shell
 CATALINA_OUT="$CATALINA_BASE"/logs/catalina.%Y-%m-%d.out
```

<br/>

### 定时脚本删除

新建脚本 vim tomcat_log_auto_del.sh 

注意该脚本的位置是假设你放在tomcat的bin中

```shell
cd `dirname $0` #返回这个脚本文件放置的目录

cd ../logs  #进入你tomcat的logs下,当然你也可以直接用绝对路径

d=`date -d '30 day ago' +%Y-%m-%d` 

rm -rf catalina.${d}.out
```



crontab -e //编辑当前用户的crontab文件，可指定具体的用户

加入我们的定时任务(表示每天3点跑一次脚本)：

```bash
0 3 * * * /usr/local/tomcat_log_auto_del.sh >/dev/null 2>&1 
```

<br/>



### 另外一种解决思路：

用shell定时脚本来复制， cp 当天的日志 指定日期命名 来复制当天的日志

清空 catalina.out 的内容，echo "" >  

随便可以删除1个月以前的日志

注意该脚本的位置是假设你放在tomcat的bin中

```bash
#!/bin/bash 
cd `dirname $0`  #返回这个脚本文件放置的目录 
d=`date +%Y-%m-%d` 
d30=`date -d '30 day ago' +%Y-%m-%d`  #输出30天前的日期 
cd ../logs/ #从bin -> logs 
cp catalina.out catalina.${d}.out
echo "" > catalina.out # 清空日志 
rm -rf catalina.${d30}.out #删除30天前的日期对应的日志 
```

<br/>

### 参考博文：

http://www.mrliangqi.com/909.html

https://blog.csdn.net/fly910905/article/details/78528652