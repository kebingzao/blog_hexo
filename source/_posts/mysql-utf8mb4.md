---
title: 如何在mysql数据库中保存 emoji 特殊字符?
date: 2019-09-27 21:20:14
tags: mysql
categories: mysql遇到的小问题
---
## 前言
之前遇到一个问题，我试图在 mysql 中保存一个 emoji 字符，但是发现取出来之后是不对的，也没法显示？？？
我查了一下，我这个库的字符集确实是 `utf8`, 所以应该是可以保存 emoji 标签这种4个字节的字符的。但是我后面查了一下资料，发现根本不是我想的那样。 mysql 的字符集 `chareacter-set` 中的 `utf8` 根本不是我理解的 `UTF-8`, mysql 的 `utf8`只支持每个字符最多三个字节，而真正的 `UTF-8` 是每个字符最多四个字节。MySQL一直没有修复这个bug，他们在2010年发布了一个叫作 `utf8mb4` 的字符集，绕过了这个问题, `mb4`就是`most bytes 4`的意思，专门用来兼容四字节的`unicode`。
所以总结了一下这个问题：
1. MySQL的`utf8mb4` 是真正的`UTF-8`
2. MySQL的`utf8` 是一种 `专属的编码`，它能够编码的 Unicode 字符并不多

<!--more-->
## 解决
既然知道问题了，那么解决就很简单了，将 库，表，列 的编码都设置为 `utf8mb4` 就行了：
```mysql
   ALTER DATABASE database_name CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
   ALTER TABLE table_name CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   ALTER TABLE table_name CHANGE column_name VARCHAR(191) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```
然后重启一下 mysql 服务。 同时在程序中，设置连接集的时候，也要设置 charset 为 `utf8mb4` (原先是 `utf8`), 以 php 为例：
```php
'db' => [
            'class' => 'yii\db\Connection',
            'dsn' => env('DB_DSN'),
            'emulatePrepare' => true,
            'username' => env('DB_USER'),
            'password' => env('DB_PASSWORD'),
            'charset' => 'utf8mb4',
        ],
```
这样就可以了
## 总结
所有在使用`utf8`的MySQL的用户都应该改用`utf8mb4`，永远都不要再使用`utf8`.












