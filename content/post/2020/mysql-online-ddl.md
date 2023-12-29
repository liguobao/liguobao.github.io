---
layout: post
title: MySQL Online DDL导致全局锁表案例分析
category: MySQL
date: 2019-05-11
tags:
- MySQL
- online ddl
---

# MySQL Online DDL导致全局锁表案例分析

## 我这边遇到了什么问题?

线上给某个表执行新增索引SQL, 然后整个数据CPU打到100%, 连接数暴增到极限, 最后导致所有访问数据库的应用都奔溃.

SQL如下:

```SQL
ALTER TABLE `book` 
ADD INDEX `idx_sub_title` (`sub_title` ASC);
```

能看到什么?

![tu1](https://ws1.sinaimg.cn/large/64d1e863gy1g2xs6mvu97j21aw05ygof.jpg)

```log
'10063293', 'root', '10.0.0.1:35252', 'novel', 'Query', '50', 'Waiting for table metadata lock', 'ALTER TABLE `lemon_novel`.`book` \nADD INDEX `idx_sub_title` (`sub_title` ASC)'


'10094494', 'root', '172.16.2.112:42808', 'novel', 'Query', '31', 'Waiting for table metadata lock', 'SELECT \n            book_trend.book_id AS book_id,
   
```

很奇怪, 这两边都在等"Waiting for table metadata lock"

## 反手查一下"Waiting for table metadata lock"是什么

1. [MySQL出现Waiting for table metadata lock的原因以及解决方法](https://www.cnblogs.com/digdeep/p/4892953.html)

2. [mysql: Waiting for table metadata lock](https://zhuanlan.zhihu.com/p/30551926)

3. [How do I find which transaction is causing a “Waiting for table metadata lock” state?](https://stackoverflow.com/questions/13148630/how-do-i-find-which-transaction-is-causing-a-waiting-for-table-metadata-lock-s)

4. [MySQL:8.11.4 Metadata Locking](https://dev.mysql.com/doc/refman/5.7/en/metadata-locking.html)

5. [MySQL:14.13.1 Online DDL Operations](https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl-operations.html)


## 初步的一些结论


看下来下面的一些结论:

1. MySQL 5.6以后的版本，支持在线DDL，新增index/删除index之类的可以直接InPlace操作，不需要rebuild整张表，理论上效果是很快的，详细资料见Online DDL Operations

2. DDL add index 操作会lock table metadata，此操作是导致我们服务不可用的原因

3. 有怀疑过lock tabel matadata和MySQL autocommit有关，但是实践下来两者看起来没有关联。

后来在阿里云上面还看到过他们特定写过类似的答疑.

1. [解决MDL锁导致无法操作数据库的问题](https://help.aliyun.com/document_detail/94566.html?spm=a2c4g.11186623.2.7.2a504335H3f8Wz#task-csn-5tt-4fb)

2. [RDS for MySQL Online DDL 使用](https://help.aliyun.com/knowledge_detail/41733.html?spm=a2c4g.11186623.4.2.2a504335nWEjej)

阿里云建议主要是这样操作.

- 这里需要找到的是一直在占用该表的会话，而不是正在等待MDL锁解除的会话，注意区分。可以根据State列的状态和Info列的命令内容来进行分析判断。

- 您也可以用如下命令查询长时间未完成的事务，如果导致阻塞的语句的用户与当前用户不同，请使用导致阻塞的语句的用户登录来终止会话。

```SQL
select concat('kill ',i.trx_mysql_thread_id,';') from information_schema.innodb_trx i,
  (select 
         id, time
     from
         information_schema.processlist
     where
         time = (select 
                 max(time)
             from
                 information_schema.processlist
             where
                 state = 'Waiting for table metadata lock'
                     and substring(info, 1, 5) in ('alter' , 'optim', 'repai', 'lock ', 'drop ', 'creat'))) p
  where timestampdiff(second, i.trx_started, now()) > p.time
  and i.trx_mysql_thread_id  not in (connection_id(),p.id);

  ```

然而在我的场景, 上面的SQL并没有任何的进程输出.

## 陷入僵局的...

不过上面给了一些思路, 现在我们主要是因为有东西占用着 table metadata lock, 导致当前所有的东西都没有执行.

show full processlist;

看一眼没什么卵用, 处理那两个奇怪的wait lock, 其他的都挺正常的.

那么, 看下现在谁占用着锁?

怎么看呢?

```SQL
select * from information_schema.innodb_trx;
```
![](https://ws1.sinaimg.cn/large/64d1e863gy1g2xsmc2ahkj21rk0d6jz8.jpg)

神奇了, 真有两个东西在占用锁.

那kill 了他们看看.

![](https://ws1.sinaimg.cn/large/64d1e863gy1g2xspdta37j21ag0bswl9.jpg)

额, 解决了.

## 最终结论

某个奇怪的程序开了查询或者奇怪的操作, lock了 table metadata, 之后连接一直都没有被释放, 导致以上各种问题.

现在的问题来了, 究竟是哪个程序或者哪个代码导致的呢?

抱歉, 我现在也还不知道...

理论上可以查, 但是上次去查的时候发现数据库显示的host对应机器的端口早就没东西了, 死无对证ing.

## 最后建议

- online DDL前,最好确认一下当前数据库有没有类似lock存在

- 最好的方案还是主从切换来搞

全文完.