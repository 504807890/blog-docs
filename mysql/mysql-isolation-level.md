+++
title = "Mysql 事务隔离等级"
author = "hao.wu"
github_url = "https://www.sorrycloud.top"
head_img = ""
created_at = 2019-11-19T17:33:32
updated_at = 2019-11-19T17:33:32
description = "当多个线程都开启事务操作数据库中的数据时，数据库系统要能进行隔离操作，以保证各个线程获取数据的准确性， 所以， 对于不同的事务，采用不同的隔离级别会有不同的结果。"
tags = ["mysql"]
+++

> Mysql 四个隔离等级 RU,RC,RR,S

##  事务隔离等级
#### 1.未提交读（Read Uncommitted）
```
一个事务能够读取到 别的事务中没有提交的更新数据。
事务可以读取到未提交的数据，这也被称为脏读(dirty read)。
```

#### 2.提交读（Read Committed）
```
一个事务只能读取到别的事务提交的更新数据。
```

#### 3.可重复读（Repeated Read）
```
保证同一事务中先后执行的多次查询将返回同意结果，不受其他事务的影响。
这种隔离级别可能出现幻读。（mysql默认的）。
```

#### 4.序列化（Serializable）
```
不允许事务并发执行，强制事务串行执行，就是在读取的每一行数据上都加上了锁，读写相互都会阻塞。
这种隔离级别最高，是最安全的，性能最低，不会出现脏读，不可重复读，幻读，丢失更新。
```

##  如何设置隔离等级
```mysql
/*设置提交读隔离级别*/
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
/*设置序列化隔离级别*/
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```