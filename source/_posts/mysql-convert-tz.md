---
title: mysql 查询的时候时区转换为本地时区
date: 2020-09-04 10:51:09
tags: mysql
categories: mysql遇到的小问题
---
## 问题
最近在做业务的时候，有遇到了一个问题，就是用户的连接记录在库里面存放的是 UTC 时间，但是在前端显示的时候，却要根据用户的本地时间来显示对应的数据。

举个例子，比如用户在前端选择查询 `2020-08-20` 和 `2020-08-25` 这段时间的连接记录。 假设这个用户当前用的是 北京时间， 也就是东八区，那么这时候， 在 mysql 进行查询的，应该就是查询 `2020-08-19 16:00:00` 和 `2020-08-24 16:00:00` 这段时间的连接记录, 因为减掉 8 小时，才是库里面真正存的 UTC 时间。

所以具体的连接记录展示是没问题的。 但是还有另一个需求， 就是用户想知道 `2020-08-20` 和 `2020-08-25` 这段时间每天连接了多长时间， 这个就不太好处理了，因为涉及到 `group by` date, 而这个 date 字段在库里面存放其实是 UTC 时间，那么 `group by` 其实只是按照 UTC 时区来的，而不是按照本地时区的 GMT 时间。 所以我们要先把它转换为 GMT 时区，然后再 `group by`。
<!--more-->
## CONVERT_TZ
mysql 有一个函数用来转换时区，`CONVERT_TZ(dt,from_tz,to_tz)`, 转换datetime值dt，从 from_tz 由给定转到 to_tz 时区给出的时区，并返回的结果值。 如果参数无效该函数返回NULL。例如:
```text
mysql> SELECT CONVERT_TZ('2004-01-01 12:00:00','+00:00','+10:00');
+---------------------------------------------------------+
| CONVERT_TZ('2004-01-01 12:00:00','+00:00','+10:00')     |
+---------------------------------------------------------+
| 2004-01-01 22:00:00                                     |
+---------------------------------------------------------+
1 row in set (0.00 sec)
```

这样子就转换成功了。

## 实操
所以针对上面的操作，就可以通过这种方式先把记录查出来，然后转换为 GMT 时区，最后再从 GMT 记录中再去 `group by`:
```text
select sum(connect_duration), date(a.start_date_gmt) as cdate from (

  select *, DATE_FORMAT(CONVERT_TZ(`start_date`,'+00:00','+08:00'), '%Y-%m-%d %H:%i:%s') as start_date_gmt
  from connect_logs 
  where start_date >= DATE_SUB('2020-08-20 00:00:00', INTERVAL 1 DAY) 
  and start_date < DATE_ADD('2020-08-25 00:00:00', INTERVAL 1 DAY)

) as a where a.start_date_gmt > '2020-08-20 00:00:00'  and  a.start_date_gmt < '2020-08-25 00:00:00' GROUP BY cdate
```

这边要注意一个细节，因为要先把 GMT 涉及到的连接记录都先跑出来，但是因为有正负12个时区，所以 查询的 UTC 时区范围，应该是 开始时间的前一天 （不一定非要减一天，可以只减 12 小时）， 和结束时间的后一天 (不一定非要加一天，可以只加 12 小时)， 然后得到的结果集肯定是包含用户本地时区的连接记录，然后用 GMT 字段保存成子表，最后再通过 GMT 的时间字段去查询并排序就可以了。







