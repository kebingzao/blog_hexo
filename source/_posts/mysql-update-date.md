---
title: mysql 设置 update 字段，每次有更新的时候，自动更新该字段
date: 2019-09-28 18:22:47
tags: mysql
categories: mysql遇到的小问题
---
## 问题
在创建表的时候，我们一般都会建一个 `create_date`， 每次代码插入的时候，不需要代码赋值， mysql 会在创建的时候，给他赋值。设置的格式是这样子的:
```mysql
 `create_date` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
```
但是如果我们想监控这一行记录什么被修改的话，就要设置另一个字段，比如 `update_date`， 一旦记录有被 update，那么就更新该字段的值，不需要程序去记录，而是由mysql自动更新。
## 解决
设置成这样子就可以了:
```mysql
`update_date` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
```

