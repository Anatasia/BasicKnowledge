包括数据定义语言DDL，数据操纵语言DML，数据控制语言DCL。

## 1.数据定义

create，drop，alter

### 1）模式定义

create schema S-T authorization wang 为用户wang定义模式S-T

drop schema &lt;模式名&gt;&lt;cascade\|restrict&gt;

cascade级联：删除模式时，把该模式中所有数据库对象全部删除

restrict限制：若该模式定义了下属的数据库对象（如表、视图等），则拒绝删除语句的执行

### 2）表定义

create table &lt;表名&gt;（列名 数据类型 完整性约束，...，foreign key &lt;属性名&gt; references &lt;表名&gt;（属性名））

alter table &lt;表名&gt;【add &lt;新列名&gt;&lt;数据类型&gt;【完整性约束条件】，drop &lt;完整性约束名&gt;，alter column&lt;列名&gt;&lt;数据类型&gt;】

### 3）视图定义

虚表，从一个或几个基本表（或视图导出的表），只存放视图的定义，不存放数据。

create view &lt;视图名&gt;（列名,...） AS &lt;子查询&gt; with check option，该语句中的子查询不允许有order by 子句或distinct

更新视图的限制：一些视图是不可更新的，因为对这些视图的更新不能唯一有意义的转换成对相应基本表的更新。

### 4）索引定义

目的：加快表的查询速度

DBMS一般会自动建立PRIMARY key 和 unique列的索引

索引是关系数据库的内部实现技术，属于内模式范畴。

create index 定义索引时可以定义索引为唯一索引、非唯一索引或聚簇索引。

create \[unique\|claster\] index &lt;索引名&gt; on &lt;表名&gt;（列名，列名）

一个基本表上只能建立一个聚簇索引，在经常查询的列上建立聚簇索引以提高查询效率，经常更新的列上不宜建立聚簇索引。

## 2.数据操纵

insert，update，delete

插入：insert into &lt;表名&gt; 【属性列】 values\[常量\]

修改：update ...set...

删除：delete from ...

## 3.数据控制

revoke，grant

## 4.数据查询

select

## 1）普通查询

a）消除取值重复的行：distinct 关键词，缺省为ALL。

b）确定范围：between ... and...，not between ... and....。

c）确定集合：in&lt;值表&gt;，not in &lt;值表&gt;

d）字符匹配：【not】like ‘&lt;匹配串&gt;’ 【escape ‘换码符’】，换码字符将通配符转义为普通字符。  '%'匹配多个字符，'\_'匹配单个字符。

e）空值：is null或is not null

f）排序 order by：可以按一个或多个属性序列排序，升序ASC，降序DESC。

g）聚集函数：COUNT，SUM， AVG， MAX， MIN。GROUP BY可以细化聚集函数的作用对象。Having 作用于组，从中选择满足条件的组。

### 2）集合查询

a）并Union：union自动去掉重复元组，union all保留重复元祖。

b）交INTERSECT

c）差EXCEPT

参加集合操作的各查询结果列数必须相同，对应项的数据类型也必须相同。

### 3）连接查询

a）等值连接

b）自然连接

c）自身连接：需要给表起别名以示区别

d）外连接：与普通连接的区别：普通连接只输出满足连接条件的元组，外连接以指定表为连接主体，将主体表中不满足连接条件的元组一并输出。

### 4）嵌套查询

a）不相关子查询

由里向外逐层查询。

b）相关子查询

子查询的条件依赖于父查询。主要分为：带有IN谓词的子查询、带有比较运算符的子查询、带有any 或all 的子查询、带有Exists的子查询。

