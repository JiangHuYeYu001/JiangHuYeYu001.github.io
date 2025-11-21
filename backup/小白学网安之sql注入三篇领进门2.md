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
（hotel,information_schema,mysql,performance_schema）。可以理解为：schemata 表是“数据库的清单”，schema_name 是清单上的“数据库名称”
。
五、table_schema：定位表所属数据库的“标识字段”
table_schema 是 information_schema.tables 和 information_schema.columns 表中的一个字段，用于表示当前记录（表或列）所属的数据库名。
它的作用类似于“标签”：
在 information_schema.tables 中，table_schema='hotel' 表示“这是 hotel 数据库下的表”；
在 information_schema.columns 中，table_schema='mysql' 表示“这是 mysql 数据库下的列”。
通过 where table_schema='目标数据库名'，可以精准过滤出你关心的数据库中的表或列，避 免查询到其他无关数据库的信息。
六、information_schema.columns：存储“所有列名”的表
information_schema.columns 存储了MySQL中所有列（字段）的信息，关键字段有：
table_schema：列所属的数据库名；
table_name：列所属的表名；
column_name：列的名称（字段名）。
作用：枚举某个表下的所有列
当你知道数据库名和表名后（比如 mysql 数据库的 user 表），可以通过 where table_schema=
'数据库名' and table_name='表名' 过滤，查询该表下的所有列。
你的payload union select 1,group_concat(column_name),3,... from information_schema.columns where table_name= 'user' 就是查询 user 表的所有列，得到 Host,User,Password,... 等字段（这些是MySQL用户表的核心字段，包含登录账号和密码）。
（这篇有点长。等下继续更新）
