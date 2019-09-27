---
title: mysql 查询有 float 类型的字段查不到 ?
date: 2019-09-27 20:48:18
tags: mysql
categories: mysql遇到的小问题
---
## 前言
之前在用 mysql 查询有 `float` 字段的时候，比如某一个字段 fee，明明有一条是  fee 为 1.99 的记录，但是就是查不到？？
```mysql
select * from fee_mode where is_pp_recurring = 1 and is_enable = 1 and fee = 1.99
```
![1](1.png)
后面查了一下，发现原来查询 `float` 字段是有坑的：
原来在 MYSQL 中，字段类型为 `float` 的字段，如果不指定 `float` 的长度和小数点位数，要根据 `float` 字段的值精确查找，结果会是空，原因是在 mysql 中，float是浮点数，Mysql存储的时候是近似值，所以用精确查找无法匹配；但可以用like去匹配。
<!--more-->
## 解决方法
### 修改为 double 类型
将`float` 改为 `double` 类型,不会出现这种问题，但是如果数据库中数据量庞大,或者修改量太大,则不适合这个方法.这个方法只适合设计数据库的初期阶段。
原因是这样的：`10.28`,这样的浮点值在电脑存放为 `10.27999973297119140625` 这种形式，同时务必注意：修改表字段类型，需要在当前表中无数据的时候修改，如果有数据的话，那已有浮点数据将会出现很多小数点。

### 设置float的精度
如果提前知道要查询的 float 字段的值的精度的话，只要设置float的精度然后进行查询也是可以的，比如本例中，1.99 就是两个精度，那么就可以这样查询：
```mysql
select * from fee_mode where format(fee,2) = format(1.99,2);
```
这样也是可以的。

### 使用concat函数
使用了concat函数，将浮点数用空字符串连接起来，转换成字符串来进行查询：
```mysql
select * from fee_mode where is_pp_recurring = 1 and is_enable = 1 AND concat(fee, '') = '1.99'
```
![1](2.png)

我们用的就是这种方法。
