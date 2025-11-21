”sql注入，狗都不学，实战能有啥用？“
切勿好高骛远。sql注入的思想是web渗透的基础。类似的思路通用在前端测试，真实渗透场景之中。而且谁知道内网有多少sql注入的漏洞呢（edusrc,说的就素你）
绝大多数web安全的基础都是代码审计。任何一个漏洞都要对其原理了解透彻。sql注入是基于对mysql数据库的一定了解和掌握而来。作为一个尽职尽责的小白向教程，笔者将在另一篇中讲述网站搭建与mysql入门作为本篇的前置。
我们从一个简单的场景开始。在2003年的第一场雪飘落时，小明正上网查博客，发现了一个url:
https://website.edu/blog?id=1
小明不懂得什么是网络安全，但是作为网页开发者，他猜想后端处理博文查询的语句是这样的：
SELECT * from blog where id=1 and private=0 LIMIT 1;
在 MySQL 中，LIMIT 1 是 LIMIT 语法的简化形式，核心作用是 只返回查询结果中的前 1 条记录。
LIMIT 有两种常用格式：
LIMIT  行数 ：省略 “偏移量” 时，默认偏移量为 0（即从第 1 条记录开始），表示“返回前 N 条记录”，（如小明猜想的LIMIT 1）；
LIMIT 偏移量, 行数：即之前讲的 “跳过前 X 条，返回 Y 条”（如 LIMIT 1,1）
因此，LIMIT 1 等价于 LIMIT 0, 1，表示：
偏移量为 0  → 从第 1 条记录开始（不跳过任何记录）；
行数为 1  → 只返回 1 条记录。
上面这条SQL语句正在从blog表中查找一条记录：该记录对应的文章id编号为1，且private列的值设为0（private列值为0表示该文
章可公开查看），同时该语句将查询结果限制为仅返回一条匹配记录。
小明随手把id改成2，url变成https://website.uirer/blog?id=2
可恶，站长不让我看，弹出403，莫非有什么独家私藏？
猜想到后端的样子，小明兴致勃勃的尝试了一下把url小小的改造：
https://website.thm/blog?id=2;--
也就是id传为2；--这个东西。
2003年的此时此刻，博客的站长心地善良，丝毫没有对攻击者有什么防范，没有过滤掉;和-这种特殊字符。
于是后端语句变成了
SELECT * from blog where id=2;-- and private=0 LIMIT 1;
URL中的分号表示SQL语句的结束，而两个连字符会使其后的所有内容都被视为注释。实际
上，通过这种操作，你运行的查询就只是：
SELECT * from blog where id=2;--
不管是不是私密的，小明都发现了id=2的博文，在无意识之中完成了一次黑客攻击。2003年的互联网大量充斥着如此简单机械的漏洞。小明上报给了站主，获得站主50元的感谢。
多年以后，面对全国网安大会汇报厅中黑压压的人群，小明想起了那个下午。2003年的冬天飘着雪，他随手打下了人生中第一个sql注入。
当然，本文叫做一篇领进门，不能停在这里。
小明执行的只是一种名为“带内SQL注入”（In-Band SQL Injection）的SQL注入漏洞示例；此类漏洞共分为三种类型，即带内注入、盲注（Blind）和带外注入（Out-of-Band），我们将在后续对其进行讨论。
闭合
笔者开头说，sql注入的思想是web渗透的基础。类似的思路通用。闭合就很典型。
在SQL注入中，“闭合”是突破应用程序原有SQL语句结构、使恶意注入代码被数据库正确解析执行的核心手段，其本质是通过构造特殊输入，打破原始SQL中用于界定字符串、参数或逻辑块的边界符号（如单引号、双引号、括号等），从而将注入的恶意代码“嵌入”到正常SQL语句中，成为可执行的指令。
废话不多说，直接上：
SELECT * FROM users WHERE username ='用户输入'
小明随便敲一下，输入为admin' OR '1'='1
拼接后的SQL会变成：
SELECT * FROM users WHERE username ='admin' OR '1'=1‘。此时，用户输入中的第一个'闭合了原始SQL中左侧的单引号，后续的OR '1'='1则成为有效的SQL逻辑，导致查询条件判断恒真，实现注入。
闭合的核心是“匹配并打破原始边界”，具体方法取决于原始SQL中使用的边界符号，常见场景包括：
单引号闭合
最常见的场景，原始SQL用单引号包裹用户输入（如字符串参数）。例如：
SELECT * FROM goods WHERE id ='用户输入'
用户输入1' OR 1=1 -- ，拼接后为：
SELECT * FROM goods WHERE id ='1'OR 1=1 --'。
其中，'闭合左侧单引号，OR 1=1添加恶意逻辑，-- 注释掉右侧剩余的单引号（避免语法错误）
双引号闭合
SELECT * FROM users WHERE name ="用户输入"4
用户输入test" UNION SELECT 1,2,3 --
拼接后为：
SELECT * FROM usersWHERE name ="test" UNION SELECT 1,2,3 --"
括号闭合
SELECT * FROM logs WHERE (id ='用户输入')
用户输入 1') OR 1=1 --
拼接后为：
SELECT * FROM logs WHERE (id ='1')OR 1=1 --')
混合符号闭合
复杂场景中可能存在多种符号组合（如引号+括号），例如：
SELECT * FROM user WHERE (name ="用户输入") AND status=1
用户输入admin") OR 1=1 --
拼接后为：
SELECT * FROM user WHERE (name="admin") OR 1=1 --") AND status=1
闭合是SQL注入的“敲门砖”——它通过打破原始SQL的边界规则，将用户输入从“数据”转化为“可执行的SQL指令”。没有正确的闭合，所有注入逻辑都无法被数据库解析，注入也就无从谈起。因此，理解闭合的原理、掌握不同场景下的闭合技巧，是分析和利用SQL注入漏洞的核心能力。
类型判断：字符/数字
在SQL注入中，字符型注入和数字型注入的核心区别在于用户输入在原始SQL语句中是否被字符边界符号（如单引号、双引号）包裹，这直接决定了注入的构造方式和判断方法。
两者最根本的区别，决定了注入的“起点”。
例如，查询用户ID的场景，数字型注入原始SQL可能是：
SELECT * FROM users WHERE id = 用户输入; --
输入直接作为数字，无引号。
当用户输入为1时，SQL为
SELECT * FROM users WHERE id = 1;；
输入为2时，SQL为
SELECT * FROM users WHERE id = 2;。
对于字符型注入，有
SELECT * FROM users WHERE username ='用户输入'; -- 输入被单引号包裹
当用户输入为admin时，SQL为
SELECT * FROM users WHERE username ='admin';；
若被双引号包裹，则是
SELECT * FROM users WHERE username ="admin";。
由于原始SQL结构的差异，两者的注入逻辑和闭合需求完全不同。
例如，针对数字型SQL SELECT * FROM users WHERE id = 用户输入;：注入输入1 OR 1=1，拼接后SQL为SELECT * FROM users WHERE id = 1 OR 1=1;（条件恒真，返回所有用户）；注入输入1 UNION SELECT 1,version(),3，拼接后可执行联合查询，获取数据库版本。
字符型注入：
必须先“闭合”原始SQL中的字符边界（如单引号），否则注入的逻辑会被当作字符串的
一部分，无法执行。
例如，针对字符型SQL SELECT * FROM users WHERE username ='用户输入';：
若直接输入admin OR 1=1，拼接后SQL为SELECT * FROM users WHERE username ='admin OR 1=1';（OR 1=1被当作字符串，无意义）；
正确做法是先闭合单引号：输入admin' OR '1'='1，拼接后SQL为SELECT * FROMusers WHERE username='admin' OR '1'='1';（OR '1'='1成为有效逻辑，条件恒真）。
当输入特殊字符（如单引号）时，两者的报错信息或页面响应差异明显，这是判断的关键依据。
数字型注入：
输入单引号（'）时，原始SQL会出现“数字格式错误”。
例如，输入1'，拼接后SQL为SELECT * FROM users WHERE id = 1';——数据库会报错 “无效的数字格式”（如MySQL提示You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to usenear ''1''），原因是单引号破坏了数字的语法规则。
字符型注入：
输入单引号（'）时，原始SQL会出现“字符串未闭合错误”。
例如，输入admin'，拼接后SQL为SELECT * FROM users WHERE username ='admin';——数据库会报错“未闭合的引号”（如MySQL提示You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''admin'''），原因是多了一个单引号，导致字符串边界不匹配。
如何判断类型？
核心思路是通过构造特殊输入，观察页面响应（报错信息、内容变化、状态码等），推断输入在SQL中的处理方式。具体步骤如下：
 测试单引号（最常用方法）
向输入点（如URL参数、表单字段）输入单引号（'），观察响应：
若页面出现“未闭合的引号”“字符串语法错误” 等提示（如Unclosed quotation mark），则大概率是字符型注入（输入被引号包裹，单引号导致边界混乱）；若页面出现“无效数字”“数字格式错误” 等提示（如invalid number），则大概率是数字型注入（输入被当作数字，单引号破坏了数字语法）；
若页面无明显变化（可能被程序捕获错误并隐藏），则需进一步测试。
测试逻辑表达式（辅助判断）
向输入点输入简单的逻辑判断（如AND 1=1和AND 1=2），对比两次响应：
若输入AND 1=1时页面正常（或返回结果不变），输入AND 1=2时页面异常（如无结果、报错），则大概率是数字型注入。
原因：数字型SQL中，id = 原始值 AND 1=1条件成立（返回正常），id = 原始值 AND1=2条件不成立（返回异常）。
例如，原始输入为1，注入1 AND 1=1后SQL有效，注入1 AND 1=2后SQL条件为假。
若输入AND 1=1和AND 1=2时页面无差异（均正常或均异常），则大概率是字符型注入。
原因：字符型SQL中，未闭合引号时，AND 1=1会被当作字符串的一部分，不影响SQL逻辑。例如，输入admin AND 1=1，拼接后SQL为SELECT * FROM users WHERE username ='admin AND 1=1';，逻辑未改变。
测试边界符号（针对字符型的细分）
若判断为字符型，可进一步测试具体的边界符号（单引号/双引号/括号）：
输入双引号（"），若报错“未闭合的双引号”，则是双引号包裹的字符型；输入')，若报错消失或逻辑变化，可能是( '输入' )形式的嵌套（需同时闭合引号和括
号）。
总结
判断的关键是通过单引号测试观察报错类型，结合逻辑表达式的响应差异，最终确定注入类型——这是后续构造有效注入代码的前提。




/慢慢更新完




