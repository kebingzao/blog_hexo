---
title: mysql 合并两个查询结果集为一条记录
date: 2020-04-11 15:35:13
tags: mysql
categories: mysql遇到的小问题
---
## 前言
前段时间有一个很神奇的需求，因为 cacti 的业务预警需求，我要在一个表内， 找到 5 分钟之内， 得到连接失败的去重复的设备数的数量和所有连接的去重复的设备数的数量， 然后得到 5 分钟内，按照唯一设备的失败率是多少，用一个 sql 表示。

## 处理
<!--more-->
### 分别得到各自的结果集
这个两个结果集，单独做都非常简单：
```text
SELECT
    count(DISTINCT(source_device_id)) AS fail
FROM
    streaming_log
WHERE
    node_device_type = 4
AND `status` = 0
AND create_date > DATE_SUB(NOW(), INTERVAL 5 MINUTE)
AND err_code NOT IN (
    '-1620002',
    '-1620005',
    '-1620007',
    '-1620008',
    '-1620009',
    '-1620010',
    '-1520002',
    '-1520011',
    '-1520012',
    '-1735001',
    '-1735002',
    '-1737003',
    '-1736003'
)
```

![png](1.png)

这个是失败的唯一设备数。

```text
SELECT
    count(DISTINCT(source_device_id)) AS total
FROM
    streaming_log
WHERE
    node_device_type = 4
AND create_date > DATE_SUB(NOW(), INTERVAL 5 MINUTE)
```

![png](2.png)

这个是 5 分钟之内总的设备数。

### 两个结果集合在一起
这两个结果集都非常简单，难的是我要怎么把这两个结果集合起来当做一个新的表结果。而且只有一行。然后再进行  fail/total 的计算？？？

之前有用过 union 将结果集合并起来，但是新的结果集的字段还是只有 total，而没有 fail， 而且结果变成两行了:
```text
SELECT
    count(DISTINCT(source_device_id)) AS total
FROM
    streaming_log
WHERE
    node_device_type = 4
AND create_date > DATE_SUB(NOW(), INTERVAL 5 MINUTE)

UNION all

SELECT
    count(DISTINCT(source_device_id)) AS fail
FROM
    streaming_log
WHERE
    node_device_type = 4
AND `status` = 0
AND create_date > DATE_SUB(NOW(), INTERVAL 5 MINUTE)
AND err_code NOT IN ('-1620002',
                          '-1620005',
                          '-1620007',
                          '-1620008',
                          '-1620009',
                          '-1620010',
                          '-1520002',
                          '-1520011',
                          '-1520012',
                          '-1735001',
                          '-1735002',
                          '-1737003',
                          '-1736003')
```

![png](3.png)

而且是要字段全部一样的话，使用填充字段的方式的话，那也是两行？？
```text
SELECT
    count(DISTINCT(source_device_id)) AS total, 0 as fail
FROM
    streaming_log
WHERE
    node_device_type = 4
AND create_date > DATE_SUB(NOW(), INTERVAL 5 MINUTE)

UNION

SELECT
    0 as total, count(DISTINCT(source_device_id)) AS fail
FROM
    streaming_log
WHERE
    node_device_type = 4
AND `status` = 0
AND create_date > DATE_SUB(NOW(), INTERVAL 5 MINUTE)
AND err_code NOT IN ('-1620002',
                          '-1620005',
                          '-1620007',
                          '-1620008',
                          '-1620009',
                          '-1620010',
                          '-1520002',
                          '-1520011',
                          '-1520012',
                          '-1735001',
                          '-1735002',
                          '-1737003',
                          '-1736003')
```

![png](4.png)

那么怎么才能变成一行呢？ 后面真的发现了一种骚操作， 因为既然可以设置 0 作为新的字段的值，那么将前一个结果集作为新的字段的默认值，不就可以变成一行了？？
```text
SELECT
    count(DISTINCT(source_device_id)) AS total,
    (
        SELECT
            count(DISTINCT(source_device_id)) AS fail
        FROM
            streaming_log
        WHERE
            node_device_type = 4
        AND `status` = 0
        AND create_date > DATE_SUB(NOW(), INTERVAL 5 MINUTE)
        AND err_code NOT IN (
            '-1620002',
            '-1620005',
            '-1620007',
            '-1620008',
            '-1620009',
            '-1620010',
            '-1520002',
            '-1520011',
            '-1520012',
            '-1735001',
            '-1735002',
            '-1737003',
            '-1736003'
        )
    ) AS fail
FROM
    streaming_log
WHERE
    node_device_type = 4
AND create_date > DATE_SUB(NOW(), INTERVAL 5 MINUTE)
```

![png](5.png)

哈哈，果然变成一行了，接下来再对这个结果集当做新的表然后再进行处理，就可以得到失败率了。

```text
SELECT
    fail / total AS rate
FROM
    (
        SELECT
            count(DISTINCT(source_device_id)) AS total,
            (
                SELECT
                    count(DISTINCT(source_device_id)) AS fail
                FROM
                    streaming_log
                WHERE
                    node_device_type = 4
                AND `status` = 0
                AND create_date > DATE_SUB(NOW(), INTERVAL 5 MINUTE)
                AND err_code NOT IN (
                    '-1620002',
                    '-1620005',
                    '-1620007',
                    '-1620008',
                    '-1620009',
                    '-1620010',
                    '-1520002',
                    '-1520011',
                    '-1520012',
                    '-1735001',
                    '-1735002',
                    '-1737003',
                    '-1736003'
                )
            ) AS fail
        FROM
            streaming_log
        WHERE
            node_device_type = 4
        AND create_date > DATE_SUB(NOW(), INTERVAL 5 MINUTE)
    ) a
```

![png](6.png)

这样子就可以在一条 sql 中得到 5 分钟之内，按唯一设备来算的失败率了。 然后就可以放到 cacti 的脚本了。
