---
title: MySQL中GAP（间隙）锁的探索
date: 2019-10-07 08:46:00
tags:
- MySQL
toc: true
---





## **GAP锁模拟测试**

新建一个表 test（id int primary key,num int)

{% asset_img table.jpg table%}

select @@tx_isolation 查看当前session的事务级别 是默认的级别REPEATABLE READ
<!--more-->
<br/>

### **非索引列的命中**

**事务A开启事务**

*DELETE FROM TEST WHERE NUM = 50;*

{% asset_img a1.jpg a1%}

<br/>

**事务B开启事务**

插入ID = 2,NUM =20 堵塞

插入ID=6，NUM = 60 堵塞

插入ID=12，NUM = 120 堵塞

{% asset_img a2.jpg a2%}

当where中以非索引列作为更新，发现整个表均被锁住，**这种完全不走索引的锁方式是表锁**

<br/>

### **唯一索引的全部命中**

我将num字段作为唯一索引

{% asset_img unique_key.jpg unique_key%}

EXPLAIN 查看一下发现查询走的是唯一索引了

{% asset_img explain.jpg explain%}

<br/>

然后我们再试一下

**事务A开启事务**

*SELECT * FROM TEST WHERE NUM = 50 FOR UPDATE;*

{% asset_img b1.jpg b1%}

**事务B开启事务**

ID = 4,NUM = 40 成功

ID = 6,NUM = 60 成功

ID = 8,NUM = 70 成功

插入其他范围记录没问题

{% asset_img b2.jpg b2%}

<br/>

只有当NUM = 50的时候是不允许的，那是因为NUM为唯一索引，值不能一致的限制

{% asset_img b3.jpg b3%}



### **唯一索引的部分命中**

**事务A开启事务**

*SELECT * FROM TEST WHERE NUM IN (50,45) FOR UPDATE;*

这里的45是不存在的，所以只命中了50

{% asset_img part_b1.jpg part_b1%}

<br/>

**事务B开启事务**

INSERT INTO TEST(ID,NUM) VALUES (6,60) 

ID = 2,NUM = 20  成功

ID = 2,NUM = 29 成功

ID = 2,NUM = 30 成功 会提示唯一索引冲突 Duplicate entry '30' for key 'index_num

ID= 4,NUM = 40, 堵塞

ID = 6,NUM = 60 成功

ID = 8,NUM = 71 成功

{% asset_img part_b2.jpg part_b2%}

从而可以确定唯一索引的部分命中情况下 GAP锁的上锁范围是 左间隙（30，50) 也就是左间隙

<br/>

### **非唯一索引的全部命中**

当我改为非唯一索引时

{% asset_img c1.jpg c1%}

**事务A开启事务**

在事务A中删除NUM=50,这下就是全部命中

*DELETE FROM TEST WHERE NUM = 50;*

{% asset_img c2.jpg c2%}

<br/>

**事务B开启事务**

插入 ID = 2,NUM = 29 成功 ROLLBACK,开启事务

插入 ID = 2,NUM = 30 成功 

插入 ID=55,NUM=30 堵塞

插入 ID=55,NUM=70 成功

插入 ID = 4,NUM = 50 堵塞 ROLLBACK,开启事务

插入 ID = 4,NUM = 40 堵塞

插入 ID = 6,NUM = 60 堵塞

插入 ID=8,NUM=70 成功 ROLLBACK,开启事务

插入 ID=8,NUM=80 成功 

因此我们可以得出结论，在非唯一索引中的全部命中情况下，间隙（GAP)锁会在 索引左右间隙

这里是 （30，50],[50，70) 区间上锁。边界值 30，70 是否堵塞与主键值相关联



### **非唯一索引的部分命中**

**事务A开启事务**

DELETE FROM TEST WHERE NUM IN (45,50)

NUM = 45是不存在的，所以为部分命中

{% asset_img part_c1.jpg part_c1%}

<br/>

**事务B开启事务**

插入 ID = 4,NUM = 40 ,堵塞

插入 ID = 6,NUM = 60, 堵塞

插入 ID = 8,NUM=80 堵塞

**证明又是全表被锁住，上的是表锁**

<br/>

## **间隙锁总结：**

**对于当前读：**

### （1）对于主键索引和唯一索引的当前读

如果范围条件都命中，不会使用Gap锁，只会使用行锁。

如果范围条件部分命中或者都不命中，则使用Gap锁，锁住索引左间隙

### （2）对于非唯一索引的当前读

如果范围条件都命中，使用Gap锁，锁住索引左右间隙

如果范围条件部分命中，使用Gap锁，锁住整个表

### （3）对于不走索引的当前读

  锁住整个表，上的是表锁

