---
title: 浅谈 - 数据库开发规范 & 最佳实践
date: 2024-04-30 14:53:46
tags:
    - sql
    - mysql
categories: 
- 服务浅谈系列
---
## 前言
说是最佳实践，其实这只是数据库开发的基础规范，事实上在很多团队的实践上，还是会根据具体的团队开发而有些调整或者优化。

但是因为大部分都是比较基础的数据库开发规范，所以要先知其然，再知其所以然。因此这一份用来打基础还算可以。

> 因为我们的项目实践大部分都是 Mysql， 因此里面有很多是 Mysql 相关的，不过都是大同小异

## 1. 命名规范
### 1.1. 命名总规则
1. 所有名称的字符范围为：`A-Z,a-z,0-9和_(下划线)`。不允许使用其他字符作为名称。
2. 采用英文单词或英文短语（包括缩写）作为名称，不能使用无意义的字符或汉语拼音。
3. 名称应该清晰明了，能够准确表达事物的含义，最好可读，遵循“见名知义”的原则。
4. 命名不得超过30个字符的系统限制。
5. 所有数据库设计要写成文档

<!--more-->
### 1.2. 数据库命名规范
1. 数据库必须使用前缀。
2. 数据库名使用小写英文字母以及下划线组成。
3. 尽量说明是哪个应用或者系统在使用的。比如：
  ```text
  game_center
  game_stat
  ```
  备份数据库名使用正式库加上备份时间组成，如：
  ```text
  game_center_20140429
  game_stat_20140429
  ```

### 1.3. 表命名规范
1. 使用小写字母。
2. 不使用tab或tbl作为表前缀。
3. 表名以代表表内的内容的一个和多个名词组成，以下划线分隔。
4. 禁止使用MySQL保留字（保留字见附录）。
5. 相关应用的数据表使用同一前缀，如论坛的表使用 `cdb_` 前缀，博客的数据表使用`supe_` 前缀，前缀名称一般不超过5字符。如：
  ```text
  web_user
  web_group
  ```
  备份数据表名使用正式表名加上备份时间组成，如：
  ```text
  web_user_20140429
  web_group_20140429
  ```

### 1.4. 字段命名规范
1. 字段使用单词组合完成，使用小写字母。
2. 避免采用过于普通过于简单的名称。如： 用户表中，用户名的字段为`user_name`比`name`更好。
3. 布尔型的字段，以一些助动词开头，更加直接生动：如，用户是否有留言`has_message`,用户是否通过检查`is_checked`等。
4. 字段名为英文短语、`形容词+名词` 或 `助动词+动词` 的形式表示，大小写混合，遵循 `见名知义` 的原则。
5. 最好是带表名前缀。如：
  ```text
  web_user表的字段：
  user_id
  user_name
  user_pwd
  ```

### 1.5. 索引命名规范
1. 索引名以功能模块简称的小写字母和业务意义的小单词组成，以下划线分隔，每个名词都用小写字母。
2. 功能类型简称：
  - 主键：pk
  - 外键：fk
  - 索引：idx
  - 唯一索引：uni_

例：fk_payment_rental，idx_fk_customer_id

### 1.6. 存储过程命名规范
1. 存储过程名称以有业务意义的一个和多个名词组成，以下划线分隔，每个名词都用小写字母。
2. 统一以`proc_` 作为存储过程的前缀，如：proc_inventory_in_stock
3. 申明的本地变量以尽量`v_` 作为前缀。
4. 参数尽量以 `p_` 作为前缀。
5. boolean 类型尽量以 `b_` 作为前缀。
6. 临时变量以 `@` 作为前缀。

## 2. 字段类型规范
规则：用尽量少的存储空间来存放一个字段的数据。
1. 禁止使用MySQL保留字。
2. 如无备注，则表中第一个字段一定是主键且为自动增长。
3. 如无备注，则数值类型的字段请使用 `unsigned` 属性。
4. 如无备注，则所有字段都设`not null`，并设置默认值。
5. 如无备注，所有布尔类型的字段均以 `is`、`has`、`exist`或者`can`开头，且必须设置一个默认值，并设为 `0`。
6. 所有数字类型字段，都必须设置一个默认值，并设为`0`。
7. 能用`int`的就不用 `char`或`varchar`。
8. 对于`MyISAM` 存储引擎，字符串长度相对固定的时候尽量用`char`而不是`varchar`。对于`InnoDB`存储引擎，使用`varchar`。
9. 能用`tinyint`的就不用`int`。
10. 能用`varchar(20)`的就不用`varchar(255)`。
11. 能用`varchar`的就不用`text`。
12. 时间戳字段尽量使用`int`型或使用`timestamp`，但不要使用`datetime`类型。
13. 新建的表与之前的表的字段有相似或者相同的字段，`字段的名称和类型必须相同`。
14. 每个字段的`COMMENT`必须写清楚，枚举类型必须写清楚每个值到底是什么意思。
15. 禁止存储明文密码。
16. 使用`procedure analyse`分析字段。

常用数据类型：
### 2.1. 整形
1. 整形：`bit`,`tinyint`,`smallint`,`mediumint`,`int`,`bigint`
2. 浮点型：`float`, `double`, `decimal`。 `float` 和 `double` 在计算上会有轻微的舍入误差，`decimal` 更准确。 不过使用上不建议用 浮点型类型，建议乘以固定倍数转换成整数存储，可以节省存储空间，且不会带来任何附加维护成本。

- tinyint > smallint > mediumint > int > bigint > decimal(存储空间逐渐变大，而性能却逐渐变小)。
- 自增序列类型的字段只能使用`int`或`bigint`，且明确标识出无符号类型（unsigned），当该字段超过42亿时，才使用bigint。

### 2.2. 字符型
字符型：`varchar`,`char`,`ENUM` 和`SET`，`text`

1. 字符列选择类型时，尽量不要使用text数据类型，blob 类型更是要坚决杜绝。
2. 仅当字符数超过`20000`时，可以采用text类型，且所有使用text类型的字段，**必须和原表拆分，与原表主键单独单独存储在另外一个表里**。它的处理方式决定了它的性能要低于char或者是varchar类型的处理。
3. 定长字段，建议使用char类型，不定长字段尽量使用varchar，且仅仅设定适当的最大长度，而不是非常随意的给一个很大的最大长度限定，因为不同的长度范围MySQL也会有不一样的存储处理。
4. 对于状态字段，可以采用char类型，也可以尝试使用ENUM来存放，因为可以极大的降低存储空间，而且即使需要增加新的类型，只要增加于末尾，修改结构也不需要重建表数据。
5. 如果是存放可预先定义的属性数据呢？可以尝试使用SET类型，即使存在多种属性，同样可以游刃有余，同时还可以节省不小的存储空间。

### 2.3. 日期时间
日期时间：常用`timestamp`，`date`

需要精确（年月日时分秒）的时间字段，可以使用timestamp(4字节)，datetime(8字节)，优先考虑使用timestamp；如果时间字段只需要精确到天，那就用date类型。

时间类型自动修改：
- `modify_time`：`timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP;`
  > modify_time在记录被修改的时候会被自动更新成当前的时间。
- `create_time`：`timestamp NOT NULL DEFAULT ‘1970-01-01 00:00:01’;`
  > create_time在记录被修改的时候则不必自动更新成当前的时间。

### 2.4 为什么要使用 NOT NULL：
- 对表的每一行，每个为`NULL`的列都需要额外的空间来标识。
- B树索引时不会存储`NULL`值，所以如果索引字段可以为`NULL`，索引效率会下降。
- 建议用0、特殊值或空串代替`NULL`值。

## 3. 索引使用规范
1. 单表索引数不能超过16个。
2. 总索引长度至少为256字节（MyISAM和InnoDB表前缀可以达到1000字节长）。
3. 每张表必须包含主键，主键最好使用int型。
4. 为需要关联查询的外键加上索引。
5. 不要索引`blob/text`等字段。
6. 小型表不需要索引。
7. 为大型字段使用短索引。
8. 在频繁进行排序或分组的字段上建立索引。
9. 更新频繁的表不适合创建索引。
10. 在创建复合索引时，需要考虑索引的字段顺序，过滤效果好的字段需要更靠前。
11. 使用前缀索引。
12. 在高并发环境不要使用外键，太容易产生死锁，应由程序保证约束。
13. 禁止冗余索引和重复索引。

### 3.1. 选择需要索引的列
最适合索引的列是出现在`WHERE`子句中的列，或连接子句指定的列，而不是出现在`SELECT`关键字后的选择列表中的列。

### 3.2. 使用唯一索引
对于唯一值的列，请使用唯一索引。重复值越多的列，索引效果越差。

### 3.3. 使用短索引
对大型字段进行索引，应指定一个前缀长度。

例如：如果有一个`VARCHAR(200)`的列，如果在前10个字符内，多数值是唯一的，那么就不要对整个列进行索引。对前10个字符进行索引能够节省大量索引空间，也可能会使查询更快。较小的索引涉及的磁盘I/O较少，较短的值比较起来更快。更为重要的是，对于较短的键值，索引高速缓存中的块能容纳更多的键值，因此，Mysql也可以在内存中容纳更多的值。这增加了找到行而不用读取索引中较多块的可能性。

### 3.4. 考虑索引列的顺序
列的顺序对复合索引的效率有很大影响，过滤效果越好的字段需要更靠前。所以在写 where 子句的时候，有索引列的查询条件要放在更前面。

### 3.5. 使用最左前缀
一个N列的复合索引可起N个索引的作用，因为可以利用索引中最左边的列集来匹配行。

例如：一个表的a、b、c列上有一个复合索引`idx_a_b_c`,索引中的行是按`a/b/c`的次序来存放的，则该复合索引相当于同时包含了以下三个索引：
- `idx_a`		  -> 可用于搜索a列
- `idx_a_b`		-> 可用于搜索a、b列的组合
- `idx_a_b_c`	-> 可用于搜索a、b、c列的组合

所以这种情况下，一定要保证查询条件要按照顺序来。

### 3.6. 可索引的比较类型
索引可用于 `<`、`<=`、`=`、`>=`、`>` 和 `BETWEEN` 运算，在模式具有一个直接量前缀时，索引也可用于 `LIKE` 运算，比如不以通配符 `%` 开头
> `column_name LIKE 'abc%'` 这种情况下，如果 column_name 上有索引，MySQL 可以使用该索引来加速查询，因为它可以利用索引来快速定位以 'abc' 开头的字符串

### 3.7. 不要过度索引
**索引不是越多越好，每个额外的索引都要占用额外的磁盘空间，并降低写操作的性能。**

例如：在更新表内容时，索引必须进行更新，有时可能需要重构，因此，索引越多，所花的时间越长。如果有一个索引很少利用或从不使用，那么会不必要地减缓表的修改速度。

此外，MySQL在生成一个执行计划时，要考虑各个索引，这也要费时间。创建多余的索引给查询优化带来了更多的工作。索引太多，也可能会使MySQL选择不到所要使用的最好索引。只保持所需的索引有利于查询优化。

如果想给已索引的表增加索引，应该考虑所要增加的索引是否是现有多列索引的最左前缀。如 `3.5` 所述，如果是，就不要费力去增加这个索引了，因为已经有了。

### 3.8. 更新频繁的表不适合创建索引
对于更新非常频率的表，索引的维护成本非常高，如果检索需求少，或者对检索效率要求不高的时候不建议创建索引。

## 4. 表结构基本设计
1. MyISAM 最大表尺寸可以达到16TB。
2. InnoDB表空间的最大容量为64TB。
3. varchar类型的最大有效长度由最大行大小和使用的字符集确定。整体最大长度是65532字节。
4. 单表避免过多的列。
5. 静态表的性能优于动态表。
6. 避免过多的关联。
7. 避免过度使用枚举。
8. 使用小而简单的合适数据类型，除非真实数据模型中有确切的的需要，否则应该该尽可能地避免使用NULL值。
9. 尽量使用相同的数据类型存储相似或相关的值，尤其是要在关联条件中使用的列。
10. 注意可变长字符串，其在临时表和排序时可能导致悲观的按最大长度分配内存。
11. 尽量使用整型定义标识列。

### 4.1. 信息收集
在建表的时候，业务人员应该对这个表在后续的一些表现和迭代中，有一定的预判，具体表现在以下几点:
1. **数据量**: 1~3 年内数据量会达到什么级别，每条数据大概有多少字节
2. **字段**: 是否有大字段，那些字段是否经常被更新
3. **查询**: 哪些字段经常出现在 `where`、`group by`、`order by` 中
4. **更新**: 哪些字段会经常出现在`update`或`delete`的`where`中
5. **SQL量统计比**: `select : update+delete : insert = ?`
6. **大数据量查询**: 预计大表及相关联的SQL，每天总的执行量在何数量级
7. **业务**: 更新为主的业务还是查询为主的业务
8. **并发**: 并发情况如何
9. **存储引擎**: 选择 InnoDB 还是 MyISAM

### 4.2. 字段的顺序
**字段的排序顺序**：自增列、int类型的、varchar类型的、时间类型的、状态类型的(status,is_deleted)。

**基本原则**：主要内容在前，次要内容在后，不为空的在前，可以空的在后面。

### 4.3. 字段的类型
1. 参考上述的 `字段类型规范`

  例：对于MyISAM表，最好使用固定长度的列代替可变长度的列。虽然会多占用一些空间，但固定长度的数据行被处理的速度比可变长度的数据行要快一些。对于那些频繁更新的表来说，这一点尤其突出，因为在那些情况下，性能更容易受到磁盘碎片的影响。在表崩溃的情况下，列长固定的表也更容易重新构造。

2. 数据行的存储格式应该使用最适合存储引擎的那种。

### 4.4. 字段的长度
1. 在可以使用短数据列的时候就不要用长的，而是按实际需要取值。
    > 例：如果你在数据列中存储的最长的值有40个字符，就不要定义成CHAR(255)，而应该定义成CHAR(40)。
2. 可变长字符串在临时表和排序时可能导致悲观的按最大长度分配内存。
3. VARCHAR 字段类型的最大长度受以下因素影响（以字符为单位）：
    1. **存储限制**: VARCHAR字段是将实际内容单独存储在聚簇索引之外，内容开头用1到2个字节表示实际长度（长度超过255时需要2个字节），因此最大长度不能超过65535。
    2. **编码长度限制**:
        - 字符类型若为gbk，每个字符最多占2个字节，最大长度不能超过 32766。
        - 字符类型若为utf8，每个字符最多占3个字节，最大长度不能超过 31845。
        - 若定义的时候超过上述限制，则VARCHAR字段会被强制转为TEXT类型，并产生 warning。
    3. **行长度限制**: 导致实际应用中VARCHAR长度限制的是一个行定义的长度。MySQL要求一个行的定义长度不能超过65535。若定义的表长度超过这个值，则提示：
        ```text
         ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. You have to change some columns to TEXT or BLOBs。
          例：若一个表定义为：
          create table tab(c1 int,c2 char(30),c3 varchar(N)) charset=utf8;
          则此处N的最大值为(65535-1-2-4-30*3)/3=21812
          减1的原因是实际行存储从第二个字节开始；
          减2的原因是varchar头部的2个字节表示长度；
          减4的原因是int类型的c占4个字节；
          减30*3的原因是编码是utf8的char(30)占用90个字节；
          如果长度超过上述长度，则被强制转成text类型，则每个字段占用定义长度为11字节，当然这已经不是”varchar”了。
        ```
        
4. char字段类型的最大长度为255（以字符为单位）。

### 4.5. 避免字段默认值使用null
MySQL NULL 类型和 oracle 的NULL有差异，会进入索引中，如果是一个复合索引，那么这个NULL类型的字段会极大影响整个索引的效率。此外，NULL在索引中的处理也是特殊的，也会占用额外的存放空间。

### 4.6.存储引擎的选择
1. 默认使用 MyISAM 存储引擎。
2. 需要用到事务或一些特性的使用 InnoDB 存储引擎。

MySQL的存储引擎包括：MyISAM、InnoDB、BDB、MEMORY、MERGE、EXAMPLE、NDB Cluster、ARCHIVE、CSV、BLACKHOLE、FEDERATED等

其中 InnoDB 和 BDB 提供事务安全表，其他存储引擎都是非事务安全表。

#### 4.6.1. 各种存储引擎的特性
以下为几种常用的存储引擎并对比各个存储引擎之间的区别和推荐使用方式。

| 功能 / 存储引擎 | MyISAM | InnoDB | Memory | Archive |
| :--- | :--- | :--- | :--- | :--- |
| 存储限制 | 无限制（实际上受限于文件系统） | 无限制（实际上受限于文件系统） | 有限（受限于系统内存） | 无限制（实际上受限于文件系统） |
| 事务安全 | 不支持 | 支持 | 不支持 | 不支持 |
| 锁机制 | 表级锁 | 行级锁 | 表级锁 | 行级锁 |
| B树索引 | 支持 | 支持 | 支持 | 不支持 |
| 哈希索引 | 不支持 | 仅在内存哈希索引可用时支持 | 支持 | 不支持 |
| 全文索引 | 支持 | 仅在MySQL 5.6.4及更高版本中支持 | 不支持 | 不支持 |
| 集群索引 | 不支持 | 支持 | 不支持 | 不支持 |
| 数据缓存 | 不支持 | 支持 | N/A（所有数据都存储在内存中） | 不支持 |
| 索引缓存 | 支持 | 支持 | N/A（所有索引都存储在内存中） | 不支持 |
| 数据可压缩 | 支持 | 仅在InnoDB Plugin和MySQL 5.6.4及更高版本中支持 | 不支持 | 支持 |
| 空间使用 | 高（因为每个MyISAM表都有独立的索引文件和数据文件） | 高（因为InnoDB存储引擎使用一个包含数据和索引的表空间） | 低（因为数据仅存储在内存中） | 低（因为Archive使用压缩来减少数据存储空间） |
| 内存使用 | 低（因为数据不缓存在内存中） | 高（因为InnoDB使用缓冲池来缓存数据和索引） | 高（因为所有数据都存储在内存中） | 低（因为数据不缓存在内存中） |
| 批量插入速度 | 高 | 低 | 高 | 高 |
| 支持外键 | 不支持 | 支持 | 不支持 | 不支持 |

以上信息可能根据MySQL的不同版本和配置有所不同。

#### 4.6.2. 选择合适的存储引擎
1. 如果应用不需要事务，处理的只是基本的CRUD操作，那么 MyISAM 是不二选择。
2. 如果是需要事务支持，并且有较高的并发读写频率，InnoDB是不错的选择。

### 4.7. 适当的拆分/冗余
1. 当我们的表中存在类似于`TEXT`或者是很大的`VARCHAR`类型的大字段的时候，如果我们大部分访问这张表的时候都不需要这个字段，我们就该义无反顾的将其拆分到另外的独立表中，以减少常用数据所占用的存储空间。这样做的一个明显好处就是每个数据块中可以存储的数据条数可以大大增加，既减少物理IO次数，也能大大提高内存中的缓存命中率。
2. 被频繁引用且只能通过join 2张（或者更多）大表的方式才能得到的独立小字段，这样的场景由于每次join仅仅是为了取得某个小字段的值，join到的记录又大，会造成大量不必要的IO，完全可以通过空间换取时间的方式来优化(将这个小字段各自表冗余一份)。不过，冗余的同时需要确保数据的一致性不会遭到破坏，确保更新的同时冗余字段也被更新。

### 4.8. 控制表的大小
MySQL在处理大表（char的表>500W行，或int表>1000W）时，性能就开始明显降低，所以要采用不同的方式控制单表容量。
- 根据数据冷热，对数据纵向分组存储，历史归档。
- 采用分库/分表/分区表，横向拆分控制单表容量，单表数据量尽量保持在`5000W`以内。
- 对于OLTP系统，控制单事务的资源消耗，遇到大事务可以拆解，采用化整为零模式，避免特殊影响大众。
- 单库不要超过500个表。
- 单表字段不要太多，最多不超过50个。

### 4.9. 表的定义参数
1. `ENGINE`: 根据自己的业务需要选择合适的存储引擎，一般事务表选择Innodb，只读表选择MyISAM。
2. `AUTO_INCREMENT`：自增列的初始化值。
3. `CHARSET`：根据自己业务的需求，定义表的字符集，对于多种语言环境选择utf8。(如果是 Mysql，因为 Mysql 的 utf8 是有缺陷的，要选择 utf8mb4)
4. `ROW_FORMAT`：行的存储格式，有以下类型：
    - **compact**（innodb默认行格式）: 结构：变长字段长度列表，null标志位，记录头信息，列1数据，列2数据... …
        > 这个格式当初的设计是为了能高效存放数据，同时还有两个隐藏列：事物ID列和回滚指针列。如果没有定义主键的话，每行还会增加个6字节长度的rowid列作为隐藏主键。
    - **compressed**: compressed格式存储的行数据会进行zlib算法压缩，所以适合存储 blob，text 之类的大长度类型的数据。
    - **dynamic**: dynamic格式考虑的是如果一个较长的数据的一部分需要存储在溢出页上，那么通常最有效的方式就是将所有数据存储在溢出页上。较短的列仍然会存放在Btree节点上，可以减少对任何给定行所需的最少溢出页的数量。适合动态，比如 varchar 之类的动态长度字段。
    - **fixd**: fixd行格式适合静态定长型，如 char。


### 4.10. 静态表的性能优于动态表
只要有可能，应该尽量将静态格式的数据存放在静态表中，而动态格式的数据可以存放在动态表中。
1. 静态格式是 MyISAM 表的默认存储格式。当表不包含变量长度列（VARCHAR、BLOB或TEXT）时，使用这个格式。每一行用固定字节数存储。静态格式是最快的 on-disk 格式。快速来自于数据文件中的行在磁盘上被找到的容易方式：当按照索引中的行号查找一个行时，用行长度乘以行号。
2. 如果一个 MyISAM 表包含任何可变长度列（VARCHAR、BLOB或TEXTDynamic），或者如果一个表被用`ROW_FORMAT=DYNAMIC`选项来创建，动态存储格式被使用。在每个长度大于4的记录前面是一个位图，该位图表明哪一列包含空字符串（对于字符串列）或者0（对于数字列）。注意，这并不包括包含NULL值的列。如果一个字符在被拖曳空间移除后长度为零，或者一个数字列为零值，这都在位图中标注了且列不被保存到磁盘。非空字符串被存放为一个长度字节加字符串的内容。每个动态格式的行仅使用必需大小的空间，当一个记录变大，它就按需要被分开成多片，造成记录碎片的后果。动态表在崩溃后要比静态表更难重建。

### 4.11. 创建合适索引
索引需要额外的维护成本、访问成本和空间成本，所以创建索引一定要谨慎，使单个索引尽量覆盖多的SQL，更新频率比较高的表要控制索引的数量。
1. 对于非常大更新量的数据，索引的维护成本会非常高，如果其检索需求很少，而且对检索效率没有非常高的要求的时候，并不建议创建索引 ，或者是尽量减少索引。
2. 对于数据量极小，小到通过索引检索还不如直接遍历来得快的时候，也并不适合使用索引。
3. 应该尽量让查找条件尽可能多的在索引中，尽可能通过索引完成所有过滤，回表只是取出额外的数据字段。
4. 字段的顺序对复合索引有至关重要的作用，过滤效果越好的字段需要更靠前。
5. 需要读取的数据量占整个数据量的比例较大或者说索引的过滤效果并不是太好的时候，使用索引并不一定优于全表扫描。
6. 在实际使用过程中，一次数据访问一般只能利用1个索引，这一点在索引创建过程中一定要注意，不是说一条SQL语句中where子句里面每个条件都有索引能对应上就可以了。
7. 在高并发环境下不要使用外键，很容易产生死锁，应由程序保证约束。
8. 字符字段必须使用能区分大多数数据的最短索引。

## 5. SQL语句规范
### 5.1. SQL语句书写
1. 不允许写 `SELECT * FROM … …` ，必须指明需要读取的具体字段。
2. 不允许在关键应用程序代码中直接写SQL语句访问数据库(使用 ORM(对象关系映射)工具 或者 预编译 SQL 可以防止 SQL 注入和提高程序的运行效率)。
3. 尽量少排序。
4. 尽量少join。
5. 考虑使用`exists`代替`in`,这样可以避免做一次`full table scan`。
6. `limit`语句尽量要跟`order by`或`distinct`。
7. 避免在一行内写太长的SQL语句，在SQL关键字的地方将SQL语句分解成多行会更加清晰。
    ```text
    如：
    SELECT UserID,UserName,UserPwd FROM User_Login WHERE AreaID = 20
    修改成：
    SELECT UserID,UserName,UserPwd
    FROM User_Login
    WHERE AreaID = 20
    更加直观。
    ```
8. 尽量不要在索引列上进行数据运算或函数运算。
9. 避免大SQL，拆解成多个小SQL。
10. 尽量用`in()/union`替换`or`，并注意in的个数小于300。
11. 避免使用`%前缀` 的模糊前缀查询。
12. 尽量避免过多的使用子查询，优先考虑使用join来替换。

### 5.2. SQL语句优化
#### 5.2.1. 消除对大型表行数据的顺序存取
某些形式的where子句强迫优化器使用顺序存取，如：
```text
select * from orders where (customer_num = 104 and order_num > 1001) or order_num = 1008
```
虽然在customer_num和order_num上建有索引，但是在上面的语句中优化器还是使用顺序存取路径扫描整个表。因为这个语句要检索的是分离的行的集合，所以应该改为如下语句：
```text
select * from orders where customer_num = 104 and order_num > 1001
union
select * from orders where order_num = 1008;
```
这样就能利用索引路径处理查询。

#### 5.2.2. 避免或简化排序
应当简化或避免对大型表进行重复的排序。当能够利用索引自动以适当的次序产生输出时，优化器就避免了排序的步骤。以下是一些影响因素：
- 索引中不包括一个或几个待排序的列。
- `group by` 或`order by` 子句中列的次序与索引的次序不一样
- 排序的列来自不同的表。

#### 5.2.3. 避免相关子查询
一个列的标签同时在主查询和where子句中的查询中出现，那么很可能当主查询中的列值改变之后，子查询必须重新查询一次。

查询嵌套的层次越多，效率越 低，因此应当尽量避免子查询。如果子查询不可避免，那么要在子查询中过滤掉尽可能多的行。

#### 5.2.4.使用临时表加速查询
把表的一个子集进行排序并创建临时表，有时能加速查询。它有助于避免多重排序操作，而且在其它方面还能简化优化器的工作。

注意：临时表创建后不会反映主表的修改。在主表中数据频繁修改的情况下，注意不要丢失数据。

#### 5.2.5. 使用排序来取代非顺序存取
非顺序磁盘存取是最慢的操作，表现在磁盘存取臂的来回移动。SQL语句隐藏了这一情况，使得我们在写应用程序时很容易写出要求存取大量非顺序页的查询。

有些时候，用数据库的排序能力来替代非顺序的存取能改进查询。

#### 5.2.6. 合理使用索引覆盖扫描
如果索引没有覆盖where子句中所有的列，可以考虑重写SQL语句，尽可能使用现有的索引进行索引覆盖扫描，然后再做关联操作返回所需的列。对于偏移量很大的时候，这样的效率会提升很大。如：
```text
select id,imei from device order by imei limit 1000000,100;
```
如果这个表非常大，那么查询最好改成下面的这样子:
```text
select a. id,a.imei from airdroid.device a
inner join(
select id from device order by title limit 1000000,100
) tt using(id);
```
需要说明的是，这里的id是不符合我们的设计规范的。但从查询效率上来说，这里的“延迟关联”将大大提升查询效率，它让MySQL扫描尽可能少的页面，获取需要访问的记录后再根据关联列回原表查询需要的所有列。这个技术也可以用于优化关联查询中的LIMIT语句。

#### 5.2.7. 分页查询的优化
有时候可以将limit转换为已知位置的查询，让MySQL通过范围扫描获得对应的结果。如：
```text
select * from message limit 1000000,100;
```
改为：
```text
select * from message where id>=N limit 100;
```


## 6. 附录：
### MySQL保留字：
在MySQL中，这些保留字不能用作数据库对象（如表、列、视图和用户）的名字。下面的列表包含了SQL3标准中定义的保留字，后面的是MySQL自己的保留字列表。
- ABSOLUTE,ACTION,ADD,ALL,ALLOCATE,ALTER,AND,ANY,ARE,AS,ASC,ASSERTION,AT,AUTHORIZATION,AVG
- BEGIN,BETWEEN,BIT,BIT_LENGTH,BOTH,BY
- CASCADE,CASCADED,CASE,CAST,CATALOG,CHAR,CHARACTER,CHAR_LENGTH,CHARACTER_LENGTH,CHECK,CLOSE,COALESCE,COLLATE,COLLATION,COLUMN,COMMIT,CONNECT,CONNECTION,CONSTRAINT,CONSTRAINTS,CONTINUE,CONVERT,CORRESPONDING,COUNT,CREATE,CROSS,CURRENT,CURRENT_DATE,CURRENT_TIME,CURRENT_TIMESTAMP,CURRENT_USER, CURSOR
- DATE,DAY,DEALLOCATE,DEC,DECIMAL,DECLARE,DEFAULT,DEFERRABLE,DEFERRED,DELETE,DESC,DESCRIBE,DESCRIPTOR,DIAGNOSTICS, DISCONNECT, DISTINCT,DOMAIN,DOUBLE, DROP
- ELSE,END,END-EXEC,ESCAPE,EXCEPT,EXCEPTION,EXEC,EXECUTE,EXISTS,EXTERNAL,EXTRACT
- FALSE,FETCH,FIRST,FLOAT,FOR,FOREIGN,FOUND,FROM,FULL
- GET,GLOBAL,GO,GOTO,GRANT,GROUP
- HAVING,HOUR
- IDENTITY,IMMEDIATE,IN,INDICATOR,INITIALLY,INNER,INPUT,INSENSITIVE,INSERT,INT,INTEGER,INTERSECT,INTERVAL,INTO,IS,ISOLATION
- JOIN
- KEY
- LANGUAGE, LAST, LEADING, LEFT, LEVEL, LIKE, LOCAL, LOWER
- MATCH, MAX, MIN, MINUTE, MODULE, MONTH
- NAMES,NATIONAL,NATURAL,NCHAR,NEXT,NO,NOT,NULL,NULLIF,NUMERIC
- OCTET_LENGTH,OF,ON,ONLY,OPEN,OPTION,OR,ORDER,OUTER,OUTPUT,OVERLAPS
- PARTIAL,POSITION,PRECISION,PREPARE,PRESERVE,PRIMARY,PRIOR,PRIVILEGES,PROCEDURE,PUBLIC
- READ,REAL,REFERENCES,RELATIVE,RESTRICT,REVOKE,RIGHT,ROLLBACK, ROWS
- SCHEMA,SCROLL,SECOND,SECTION,SELECT,SESSION,SESSION_USER,SET,SIZE,SMALLINT,SOME,SQL,SQLCODE,SQLERROR,SQLSTATE,SUBSTRING,SUM,SYSTEM_USER
- TABLE,TEMPORARY,THEN,TIME,TIMESTAMP,TIMEZONE_HOUR,TIMEZONE_MINUTE,TO,TRAILING,TRANSACTION,TRANSLATE, TRANSLATION, TRIM, TRUE
- UNION,UNIQUE,UNKNOWN,UPDATE,UPPER,USAGE,USER,USING
- VALUE,VALUES,VARCHAR,VARYING,VIEW
- WHEN,WHENEVER,WHERE,WITH,WORK,WRITE
- YEAR
- ZONE

这个列表是MySQL中保留字的列表。前面的列表中已经出现过的单词在这里省略掉了。
- ANALYZE,ASENSITIVE
- BEFORE,BIGINT,BINARY,BLOB
- CALL,CHANGE,CONDITION
- DATABASE,DATABASES,DAY_HOUR,DAY_MICROSECOND,DAY_MINUTE,DAY_SECOND,DELAYED, DETERMINISTIC, DISTINCTROW, DIV, DUAL
- EACH,ELSEIF,ENCLOSED,ESCAPED,EXIT,EXPLAIN
- FLOAT4,FLOAT8,FORCE,FULLTEXT
- HIGH_PRIORITY, HOUR_MICROSECOND, HOUR_MINUTE, HOUR_SECOND
- IF, IGNORE,INDEX,INFILE,INOUT,INT1,INT2,INT3,INT4,INT8,ITERATE
- KEYS,KILL
- LABEL,LEAVE,LIMIT,LINES,LOAD,LOCALTIME,LOCALTIMESTAMP,LOCK,LONG,LONGBLOB,LONGTEXT,LOOP,LOW_PRIORITY
- MEDIUMBLOB,MEDIUMINT,MEDIUMTEXT,MIDDLEINT, MINUTE_MICROSECOND,MINUTE_SECOND, MOD, MODIFIES
- NO_WRITE_TO_BINLOG
- OPTIMIZE, OPTIONALLY, OUT, OUTFILE
- PURGE
- RAID0,READS,REGEXP,RELEASE,RENAME,REPEAT,REPLACE,REQUIRE,RETURN, RLIKE
- SCHEMAS,SECOND_MICROSECOND,SENSITIVE,SEPARATOR,SHOW, SONAME,SPATIAL,SPECIFIC,SQLEXCEPTION,SQLWARNING, SQL_BIG_RESULT, SQL_CALC_FOUND_ROWS,SQL_SMALL_RESULT, SSL, STARTING, STRAIGHT_JOIN
- TERMINATED, TINYBLOB, TINYINT, TINYTEXT, TRIGGER
- UNDO,UNLOCK,UNSIGNED,USE, UTC_DATE, UTC_TIME, UTC_TIMESTAMP
- VARBINARY, VARCHARACTER
- WHILE
- X509, XOR
- YEAR_MONTH
- ZEROFILL








