# mysql数据库
## 一、MySQL框架
### 1、MySQL基架
MySQL基架大致包括如下几大模块组件：
* MySQL向外提供的交互接口（Connectors）
* 管理服务组件和工具组件(Management Service & Utilities)
* 连接池组件(Connection Pool)
* SQL接口组件(SQL Interface)
* 查询分析器组件(Parser)
* 优化器组件（Optimizer）
* 缓存主件（Caches & Buffers）
* 插件式存储引擎（Pluggable Storage Engines）
* 物理文件（File System）
![](https://i.imgur.com/d18r75s.jpg)


#### 1.1、MySQL向外提供的交互接口（Connectors）

Connectors组件，是MySQL向外提供的交互组件，如java,.net,php等语言可以通过该组件来操作SQL语句，实现与SQL的交互。

#### 1.2、管理服务组件和工具组件(Management Service & Utilities)

提供对MySQL的集成管理，如备份(Backup),恢复(Recovery),安全管理(Security)等

#### 1.3、连接池组件(Connection Pool)

负责监听对客户端向MySQL Server端的各种请求，接收请求，转发请求到目标模块。

每个成功连接MySQL Server的客户请求都会被 创建或分配一个线程，该线程负责客户端与MySQL Server端的通信，接收客户端发送的命令，传递服务端的结果信息等。

#### 1.4、SQL接口组件(SQL Interface)

接收用户SQL命令，如DML,DDL和存储过程等，并将最终结果返回给用户。

#### 1.5、查询分析器组件(Parser)
首先分析SQL命令语法的合法性，并尝试将SQL命令分解成数据结构，若分解失败，则提示SQL语句不合理。

#### 1.6、优化器组件（Optimizer）

对SQL命令按照标准流程进行优化分析。

#### 1.7、缓存主件（Caches & Buffers）

缓存和缓冲组件

#### 1.88、MySQL存储引擎

##### （1）、什么是MySQL存储引擎？

MySQL属于关系型数据库，而关系型数据库的存储是以表的形式进行的，对于表的创建，数据的存储，检索，更新等都是由MySQL存储引擎完成的，这也是MySQL存储引擎在MySQL中扮演的重要角色。

研究过SQL Server和Oracle的读者可能很清楚，这两种数据库的存储引擎只有一个，而MySQL的存储引擎种类比较多，如MyISAM存储 引擎，InnoDB存储引擎和Memory存储引擎.

MySQL之所以有多种存储引擎，是由MySQL的开源性决定的。 MySQL存储引擎从种类上来说，大致可归结为官方存储引擎和第三方存储引起。

MySQL的开源性，允许第三方基于MySQL骨架，开发适合自己业务需求的存储引擎。

##### （2）、MySQL存储引擎作用

MySQL存储引擎在MySQL中扮演重要角色，其作比较重要作用，大致归结为如下两方面：
* 作用一 ：管理表创建，数据检索，索引创建等
* 作用二 ：满足自定义存储引擎开发。

##### （3）、MySQL引擎种类
不同种类的存储引擎，在存储表时的存储引擎表机制也有所不同，从MySQL存储引擎种类上来说，可以分为官方存储引擎和第三方存储引擎。

当前，也存在多种MySQL存储引擎，如MyISAM存储引擎，InnoDB存储引擎，NDB存储引擎，Archive存储引擎，Federated存储引擎，Memory 存储引擎，Merge存储引擎，Parter存储引擎，Community存储引擎，Custom存储引擎和其他存储引擎。

其中，比较常用的存储引擎包括InnoDB存储引擎，MyISAM存储引擎和Momery存储引擎。

##### （4）、几种典型MySQL存储引擎比较
![](https://i.imgur.com/qInOGr0.jpg)

#### 1.9、物理文件（File System）
实际存储MySQL数据库文件和一些日志文件等的系统，如Linux，Unix,Windows等。

### 2、MySQL逻辑架构
![](https://i.imgur.com/Af8599U.png)
如上图所示，MySQL服务器逻辑架构从上往下可以分为三层：
第一层：处理客户端连接、授权认证等。
第二层：服务器层，负责查询语句的解析、优化、缓存以及内置函数的实现、存储过程等。
第三层：存储引擎，负责MySQL中数据的存储和提取。MySQL中服务器层不管理事务，事务是由存储引擎实现的。
MySQL支持事务的存储引擎有InnoDB、NDB Cluster等，其中 InnoDB的使用最为广泛；其他存储引擎不支持事务，如MyIsam、Memory等。

## 二、mysql的查询
### 1、查询的过程
MySQL会解析查询，并创建内部数据结构（解析树），然后对其进行各种优化，包括重写查询，决定表的读取顺序，以及选择合适的索引等。用户可以通过特殊的关键字提示（hint）优化器，影响它的决策过程。也可以请求优化器解释（explain）优化过程的各个因素，使用户可以知道服务器是如何进行优化决策的，并提供一个参考基准，便于用户重构查询和schema，修改相关配置，是应用尽可能高效运行。 优化器并不关心使用的是什么存储引擎，但存储引擎对于优化查询是有影响的。优化器会请求存储引擎提供容量或某个具体操作的开销信息，以及表数据的统计信息等。例如，某些存储引擎的某种索引，可能对一些特定的查询有优化。对于SELECT语句，在解析查询之前，服务器会先检查查询缓存（Query Cache），如果能够在其中找到对应的查询，服务器就不必再执行查询解析、优化和执行的整个过程，而是直接返回查询缓存中的结果集。
查询的过程概括如下：
* 客户端向MySQL服务器发送一条查询请求
* 服务器首先检查查询缓存，如果命中缓存，则立刻返回存储在缓存中的结果。否则进入下一阶段
* 服务器进行SQL解析、预处理、再由优化器生成对应的执行计划
* MySQL根据执行计划，调用存储引擎的API来执行查询
* 将结果返回给客户端，同时缓存查询结果
具体如图：
![](https://i.imgur.com/w1rPgx1.png)

#### （1）客户端/服务端通信协议
MySQL 的客户端/服务端通信协议是“半双工”的：在任一时刻，要么是服务器向客户端发送数据，要么是客户端向服务器发送数据，这两个动作不能同时发生。
一旦一端开始发送消息，另一端要接收完整个消息才能响应它，所以我们无法也无须将一个消息切成小块独立发送，也没有办法进行流量控制。
客户端用一个单独的数据包将查询请求发送给服务器，所以当查询语句很长的时候，需要设置 max_allowed_packet 参数。
但是需要注意的是，如果查询实在是太大，服务端会拒绝接收更多数据并抛出异常。
与之相反的是，服务器响应给用户的数据通常会很多，由多个数据包组成。但是当服务器响应客户端请求时，客户端必须完整的接收整个返回结果，而不能简单的只取前面几条结果，然后让服务器停止发送。
因而在实际开发中，尽量保持查询简单且只返回必需的数据，减小通信间数据包的大小和数量是一个非常好的习惯，这也是查询中尽量避免使用 SELECT * 以及加上 LIMIT 限制的原因之一。

#### （2）查询缓存
在解析一个查询语句前，如果查询缓存是打开的，那么 MySQL 会检查这个查询语句是否命中查询缓存中的数据。
如果当前查询恰好命中查询缓存，在检查一次用户权限后直接返回缓存中的结果。这种情况下，查询不会被解析，也不会生成执行计划，更不会执行。

MySQL将缓存存放在一个引用表（不要理解成 table，可以认为是类似于 HashMap 的数据结构），通过一个哈希值索引。
这个哈希值通过查询本身、当前要查询的数据库、客户端协议版本号等一些可能影响结果的信息计算得来。
所以两个查询在任何字符上的不同（例如：空格、注释），都会导致缓存不会命中。
如果查询中包含任何用户自定义函数、存储函数、用户变量、临时表、MySQL库中的系统表，其查询结果都不会被缓存。
比如函数NOW()或者CURRENT_DATE()会因为不同的查询时间，返回不同的查询结果。
再比如包含CURRENT_USER或者CONNECION_ID()的查询语句会因为不同的用户而返回不同的结果，将这样的查询结果缓存起来没有任何的意义。
既然是缓存，就会失效，那查询缓存何时失效呢？MySQL 的查询缓存系统会跟踪查询中涉及的每个表，如果这些表（数据或结构）发生变化，那么和这张表相关的所有缓存数据都将失效。
正因为如此，在任何的写操作时，MySQL 必须将对应表的所有缓存都设置为失效。
如果查询缓存非常大或者碎片很多，这个操作就可能带来很大的系统消耗，甚至导致系统僵死一会儿。
而且查询缓存对系统的额外消耗也不仅仅在写操作，读操作也不例外：
* 任何的查询语句在开始之前都必须经过检查，即使这条SQL语句永远不会命中缓存。
* 如果查询结果可以被缓存，那么执行完成后，会将结果存入缓存，也会带来额外的系统消耗。

基于此，我们要知道并不是什么情况下查询缓存都会提高系统性能，缓存和失效都会带来额外消耗，只有当缓存带来的资源节约大于其本身消耗的资源时，才会给系统带来性能提升。
但如何评估打开缓存是否能够带来性能提升是一件非常困难的事情，也不在本文讨论的范畴内。
如果系统确实存在一些性能问题，可以尝试打开查询缓存，并在数据库设计上做一些优化，比如：
* 用多个小表代替一个大表，注意不要过度设计。
* 批量插入代替循环单条插入。
* 合理控制缓存空间大小，一般来说其大小设置为几十兆比较合适。
* 可以通过SQL_CACHE和SQL_NO_CACHE来控制某个查询语句是否需要进行缓存。

最后的忠告是不要轻易打开查询缓存，特别是写密集型应用。如果你实在是忍不住，可以将query_cache_type设置为DEMAND。
这时只有加入SQL_CACHE的查询才会走缓存，其他查询则不会，这样可以非常自由地控制哪些查询需要被缓存。
当然查询缓存系统本身是非常复杂的，这里讨论的也只是很小的一部分，其他更深入的话题没有涉及，比如：缓存是如何使用内存的？如何控制内存的碎片化？事务对查询缓存有何影响等等。

#### （3）语法解析和预处理
MySQL 通过关键字将 SQL 语句进行解析，并生成一棵对应的解析树。这个过程解析器主要通过语法规则来验证和解析。比如 SQL 中是否使用了错误的关键字或者关键字的顺序是否正确等等。
预处理则会根据 MySQL 规则进一步检查解析树是否合法。比如检查要查询的数据表和数据列是否存在等等。

#### （4）查询优化
经过前面的步骤生成的语法树被认为是合法的了，并且由优化器将其转化成查询计划。
多数情况下，一条查询可以有很多种执行方式，最后都返回相应的结果。优化器的作用就是找到这其中最好的执行计划。
MySQL 使用基于成本的优化器，它尝试预测一个查询使用某种执行计划时的成本，并选择其中成本最小的一个。
在 MySQL 可以通过查询当前会话的last_query_cost的值来得到其计算当前查询的成本。
```
mysql> select * from t_message limit 10;
...省略结果集

mysql> show status like 'last_query_cost';
+-----------------+-------------+
| Variable_name   | Value       |
+-----------------+-------------+
| Last_query_cost | 6391.799000 |
+-----------------+-------------+
```

示例中的结果表示优化器认为大概需要做6391个数据页的随机查找才能完成上面的查询。
这个结果是根据一些列的统计信息计算得来的，这些统计信息包括：每张表或者索引的页面个数、索引的基数、索引和数据行的长度、索引的分布情况等等。
有非常多的原因会导致MySQL选择错误的执行计划，比如统计信息不准确、不会考虑不受其控制的操作成本（用户自定义函数、存储过程）。
MySQL认为的最优跟我们想的不一样（我们希望执行时间尽可能短，但MySQL值选择它认为成本小的，但成本小并不意味着执行时间短）等等。

MySQL的查询优化器是一个非常复杂的部件，它使用了非常多的优化策略来生成一个最优的执行计划：
* 重新定义表的关联顺序（多张表关联查询时，并不一定按照SQL中指定的顺序进行，但有一些技巧可以指定关联顺序）。
* 优化MIN()和MAX()函数（找某列的最小值，如果该列有索引，只需要查找 B+Tree索引最左端，反之则可以找到最大值，具体原理见下文）。
* 提前终止查询（比如：使用LIMIT时，查找到满足数量的结果集后会立即终止查询）。
* 优化排序（在老版本MySQL会使用两次传输排序，即先读取行指针和需要排序的字段在内存中对其排序，然后再根据排序结果去读取数据行，而新版本采用的是单次传输排序，也就是一次读取所有的数据行，然后根据给定的列排序。对于I/O密集型应用，效率会高很多）。

随着MySQL的不断发展，优化器使用的优化策略也在不断的进化，这里仅仅介绍几个非常常用且容易理解的优化策略，其他的优化策略，大家自行查阅吧。

#### （5）查询执行引擎
在完成解析和优化阶段以后，MySQL 会生成对应的执行计划，查询执行引擎根据执行计划给出的指令逐步执行得出结果。
整个执行过程的大部分操作均是通过调用存储引擎实现的接口来完成，这些接口被称为handler API。
查询过程中的每一张表由一个 handler 实例表示。实际上，MySQL 在查询优化阶段就为每一张表创建了一个 handler 实例，优化器可以根据这些实例的接口来获取表的相关信息，包括表的所有列名、索引统计信息等。
存储引擎接口提供了非常丰富的功能，但其底层仅有几十个接口，这些接口像搭积木一样完成了一次查询的大部分操作。

#### （6）返回结果给客户端
查询执行的最后一个阶段就是将结果返回给客户端。即使查询不到数据，MySQL 仍然会返回这个查询的相关信息，比如该查询影响到的行数以及执行时间等等。
如果查询缓存被打开且这个查询可以被缓存，MySQL 也会将结果存放到缓存中。
结果集返回客户端是一个增量且逐步返回的过程。有可能 MySQL 在生成第一条结果时，就开始向客户端逐步返回结果集了。
这样服务端就无须存储太多结果而消耗过多内存，也可以让客户端第一时间获得返回结果。
需要注意的是，结果集中的每一行都会以一个满足①中所描述的通信协议的数据包发送，再通过 TCP 协议进行传输，在传输过程中，可能对 MySQL 的数据包进行缓存然后批量发送。

从整个查询过程可以看出查询的开销主要包括：MySQL的解析、优化、锁等待、以及数据处理及存储引用的API的调用等。

### 2、select的标准执行流程
(8)  SELECT
(9)  DISTINCT
(11) <TOP_specification> <select_list>
(1)  FROM <left_table>
(3)  <join_type> JOIN <right_table>
(2)  ON <join_condition>
(4)  WHERE <where_condition>
(5)  GROUP BY <group_by_list>
(6)  WITH {CUBE ROLLUP}
(7)  HAVING <having_condition>
(10) ORDER BY <order_by_list>

* FROM：对FROM子句中的前两个表执行笛卡尔积，生成虚拟表VT1
* ON：对VT1应用ON筛选器。只有那些使<join_condition>为真的行才被插入VT2
* OUTER（JOIN）：如果指定了OUTER JOIN，保留表中未找到匹配的行将作为外部行添加到VT2，生成VT3。如果FROM子句包含两个以上的表，则对上一个联接生成的结果表和下一个表重复执行步骤1到步骤3，直到处理完所有的表为止
* 对VT3应用WHERE筛选器。只有使<where_condition>为TRUE的行才被插入VT4
* GROUP BY：按GROUP BY 子句中的列列表对VT4中的行分组，生成VT5
* CUBEROLLUP：把超组插入VT5，生成VT6。
* HAVING：对VT6应用HAVING筛选器。只有使<having_condition>为TRUE的组才会被插入VT7
* SELECT：处理SELECT列表，产生VT8。
* DISTINCT：将重复的行从VT8中移除，产生VT9
* ORDER BY：将VT9中的行按ORDER BY子句中的列列表排序，生成一个有表（VC10）
* TOP：从VC10的开始处选择指定数量或比例的行，生成表VT11，并返回给调用者

### 3、SQL中的JOIN
**4.1、join的分类**
![](https://i.imgur.com/NCw6y4p.png)
如图，sql中的join包括outer join、left join、inner join、right join，区别与含义看图就一目了然。

**4.2、join算法**
    mysql只支持一种join算法：Nested-Loop Join（嵌套循环连接），但Nested-Loop Join有三种变种：Simple Nested-Loop Join，Index Nested-Loop Join，Block Nested-Loop Join。
**原理:**
(1).Simple Nested-Loop Join：
如下图，r为驱动表，s为匹配表，可以看到从r中分别取出r1、r2、......、rn去匹配s表的左右列，然后再合并数据，对s表进行了rn次访问，对数据库开销大
![](https://i.imgur.com/oRX5p75.png)


(2).Index Nested-Loop Join（索引嵌套）：
这个要求非驱动表（匹配表s）上有索引，可以通过索引来减少比较，加速查询。
在查询时，驱动表（r）会根据关联字段的索引进行查找，挡在索引上找到符合的值，再回表进行查询，也就是只有当匹配到索引以后才会进行回表查询。
如果非驱动表（s）的关联健是主键的话，性能会非常高，如果不是主键，要进行多次回表查询，先关联索引，然后根据二级索引的主键ID进行回表操作，性能上比索引是主键要慢。
![](https://i.imgur.com/Pp2lJmY.png)


(3).Block Nested-Loop Join：
如果有索引，会选取第二种方式进行join，但如果join列没有索引，就会采用Block Nested-Loop Join。可以看到中间有个join buffer缓冲区，是将驱动表的所有join相关的列都先缓存到join buffer中，然后批量与匹配表进行匹配，将第一种多次比较合并为一次，降低了非驱动表（s）的访问频率。默认情况下join_buffer_size=256K，在查找的时候MySQL会将所有的需要的列缓存到join buffer当中，包括select的列，而不是仅仅只缓存关联列。在一个有N个JOIN关联的SQL当中会在执行时候分配N-1个join buffer。
![](https://i.imgur.com/oZ71RFB.png)

### 4、快照读和当前读
在RR级别中，虽然让数据变得可重复读（至于如何实现可重复读的后面会提到），但我们读到的数据可能是历史数据，不是数据库最新的数据。这种读取历史数据的方式，我们叫它快照读 (snapshot read)，而读取数据库最新版本数据的方式，叫当前读 (current read)。

#### 4.1、select 快照读
当执行select操作是innodb默认会执行快照读，会记录下这次select后的结果，之后select 的时候就会返回这次快照的数据，即使其他事务提交了不会影响当前select的数据，这就实现了可重复读了。快照的生成当在第一次执行select的时候，也就是说假设当A开启了事务，然后没有执行任何操作，这时候B insert了一条数据然后commit,这时候A执行 select，那么返回的数据中就会有B添加的那条数据。之后无论再有其他事务commit都没有关系，因为快照已经生成了，后面的select都是根据快照来的。

#### 4.2、当前读
对于会对数据修改的操作(update、insert、delete)都是采用当前读的模式。在执行这几个操作时会读取最新的记录，即使是别的事务提交的数据也可以查询到。假设要update一条记录，但是在另一个事务中已经delete掉这条数据并且commit了，如果update就会产生冲突，所以在update的时候需要知道最新的数据。

实现select的当前读需要手动的加锁：
* select * from table where ? lock in share mode;
* select * from table where ? for update;


## 三、mysql性能调优基础

### 1、MySQL执行计划的理解

**id**：本次 select 的标识符。在查询中每个select都有一个顺序的数值。

**select_type：**
* simple: 简单的 select （没有使用 union或子查询）
* primary: 最外层的 select。
* union: 第二层，在select 之后使用了 union。
* dependent union: union 语句中的第二个select，依赖于外部子查询
* subquery: 子查询中的第一个 select
* dependent subquery: 子查询中的第一个 subquery依赖于外部的子查询
* derived: 派生表 select（from子句中的子查询）
* table：记录查询引用的表。

**type**：以下列出了各种不同类型的表连接，依次是从最好的到最差的：
SYSTEM > CONST > EQ_REF > REF > RANGE > INDEX > ALL（不仅仅是连接，单表查询也会有）

* system：表只有一行记录（等于系统表）。这是 const 表连接类型的一个特例
* const：表中最多只有一行匹配的记录，它在查询一开始的时候就会被读取出来。由于只有一行记录，在余下的优化程序里该行记录的字段值可以被当作是一个恒定值。const表查询起来非常快，因为只要读取一次！const 用于在和 primary key 或unique 索引中有固定值比较的情形
* wq_ref:从该表中会有一行记录被读取出来以和从前一个表中读取出来的记录做联合。与const类型不同的是，这是最好的连接类型。它用在索引所有部 分都用于做连接并且这个索引是一个primary key 或 unique 类型。eq_ref可以用于在进行“=”做比较时检索字段。比较的值可以是固定值或者是表达式，表达示中可以使用表里的字段，它们在读表之前已经准备好了
* ref: 该表中所有符合检索值的记录都会被取出来和从上一个表中取出来的记录作联合。ref用于连接程序使用键的最左前缀或者是该键不是 primary key 或 unique索引（换句话说，就是连接程序无法根据键值只取得一条记录）的情况。当根据键值只查询到少数几条匹配的记录时，这就是一个不错的连接类型。 ref还可以用于检索字段使用=操作符来比较的时候。以下的几个例子中，MySQL将使用 ref 来处理ref_table，和eq_ref的区别是-用到的索引是否唯一性
* range：只有在给定范围的记录才会被取出来，利用索引来取得一条记录。range用于将某个字段和一个定值用以下任何操作符比较时：=、<>、>、>=、<、<=、is null、<=>、between、and 或 in，例子：
select * from tbl_name where key_column = 10;
select * from tbl_name where key_column between 10 and 20;
select * from tbl_name where key_column in (10,20,30);
select * from tbl_name where key_part1= 10 and key_part2 in (10,20,30);

* index：连接类型跟 all 一样，不同的是它只扫描索引树。它通常会比 all快点，因为索引文件通常比数据文件小。MySQL在查询的字段只是单独的索引的一部分的情况下使用这种连接类型
* all: 将对该表做全部扫描以和从前一个表中取得的记录作联合。这时候如果第一个表没有被标识为const的话就不大好了，在其他情况下通常是非常糟糕的。正常地，可以通过增加索引使得能从表中更快的取得记录以避免all

**possible_keys**：possible_keys字段是指 MySQL 在搜索表记录时可能使用哪个索引。注意，这个字段完全独立于 explain 显示的表顺序。这就意味着 possible_keys 里面所包含的索引可能在实际的使用中没用到。如果这个字段的值是null，就表示没有索引被用到。这种情况下，就可以检查 where子句中哪些字段那些字段适合增加索引以提高查询的性能。就这样，创建一下索引，然后再用 explain 检查一下。想看表都有什么索引，可以通过 show index from tbl_name 来看。

**key**：key字段显示了 MySQL 实际上要用的索引。当没有任何索引被用到的时候，这个字段的值就是null。想要让 MySQL 强行使用或者忽略在 possible_keys 字段中的索引列表，可以在查询语句中使用关键字 force index、 use index 或 ignore index。

**key_len**：key_len 字段显示了MySQL使用索引的长度。当 key 字段的值为 null时，索引的长度就是 null。注意，key_len的值可以告诉你在联合索引中 MySQL 会真正使用了哪些索引。

**ref**：ref 字段显示了哪些字段或者常量被用来和 key 配合从表中查询记录出来。

**rows**：rows 字段显示了 MySQL 认为在查询中应该检索的记录数。MySQL 认为的得到结果需要读取的记录数，不是返回的记录数。

**extra**：
* not exists：mysql 在查询时做一个 left join 优化时，当它在当前表中找到了和前一条记录符合 left join 条件后，就不再搜索更多的记录了。下面是一个这种类型的查询例子：select * from t1 left join t2 on t1.id=t2.id where t2.id is null;假使 t2.id 定义为 not null。这种情况下，mysql将会扫描表 t1并且用 t1.id 的值在 t2 中查找记录。当在 t2中找到一条匹配的记录时，这就意味着 t2.id 肯定不会都是 null，就不会再在 t2 中查找相同 id 值的其他记录了。也可以这么说，对于 t1 中的每个记录，mysql 只需要在t2 中做一次查找，而不管在 t2 中实际有多少匹配的记录。
* range checked for each record (index map)：mysql没找到合适的可用的索引。取代的办法是，对于前一个表的每一个行连接，它会做一个检验以决定该使用哪个索引（如果有的话），并且使用这个索引来从表里取得记录。这个过程不会很快，但总比没有任何索引时做表连接来得快。
* using filesort：mysql 需要额外的做一遍排序从而以排好的顺序取得记录。排序程序根据连接的类型遍历所有的记录，并且将所有符合 where 条件的记录的要排序的键和指向记录的指针存储起来。这些键已经排完序了，对应的记录也会按照排好的顺序取出来。
* using index：字段的信息直接从索引树中的信息取得，而不再去扫描实际的记录。这种策略用于查询时的字段是一个独立索引的一部分。
* using temporary：mysql需要创建临时表存储结果以完成查询。这种情况通常发生在查询时包含了 group by 和 order by 子句，它以不同的方式列出了各个字段。
* using where：where 子句将用来限制哪些记录匹配了下一个表或者发送给客户端。除非你特别地想要取得或者检查表种的所有记录，否则的话当查询的 extra 字段值不是 using where 并且表连接类型是 all 或 index 时可能表示有问题。如果你想要让查询尽可能的快，那么就应该注意 extra 字段的值为using filesort 和 using temporary 的情况。

### 2、profiling的使用及理解
profiling是分析查询语句非常重要的利器，设置了profiling之后，你的每一个sql语句都会被记录分析，可以看到MySQL为我们生成了操作在执行过程中每一个步骤所耗费的时间，通过profiling我们就可以查找出sql语句中耗时的地方，并加以优化。
#### 2.1、使用方式
set profiling=on; 打开profiling 。
show profiles；可以查看在打开profiling之后所有被记录的操作。
show profile for Query_ID(Query_ID是记录操作对应的ID);查看对应语句具体的执行过程分析。分析示例如下：

![](https://i.imgur.com/K6hzimv.png)

show profile all for Query_ID;和上面的类似，只是结果更加详尽。分析示例如下：
![](https://i.imgur.com/TKwqNjx.png)

#### 2.2、意义详解
**横向栏意义**
| 关键词 | 示例|含义| 
| -------- | -------- | -------- |
| Status |  query end | 状态|
| Duration |  1.751142 | 持续时间|
| CPU_user |  0.008999 | cpu用户|
| CPU_system |  0.003999 | cpu系统|
| Context_voluntary |  98 | 上下文主动切换|
| Context_involuntary |  0 | 上下文被动切换|
| Block_ops_in |  8 | 阻塞的输入操作|
| Block_ops_out |  32 | 阻塞的输出操作|
| Messages_sent |  0 | 消息发出|
| Messages_received |  0 | 消息接受|
| Page_faults_major |  0 | 主分页错误|
| Page_faults_minor |  0 | 次分页错误|
| Swaps |  0 | 交换次数|
| Source_function |mysql_execute_command| 源功能|
| Source_file |  sql_parse.cc | 源文件|
| Source_line |  4465 | 源代码行|


**纵向栏意义**
| 关键词 | 含义| 
| -------- | -------- | 
|starting|开始|
|checking permissions|检查权限|
|Opening tables|打开表|
|init | 初始化|
|System lock |系统锁|
|optimizing | 优化|
|statistics | 统计|
|preparing |准备|
|executing |执行|
|Sending data |发送数据|
|Sorting result |排序|
|end |结束|
|query end |查询结束|
|closing tables | 关闭表 ／去除TMP 表|
|freeing items | 释放物品|
|cleaning up |清理|


#### 2.3、常见的耗时过程的处理
| 状态 | 描述| 
| -------- | -------- | 
| System lock     | 确认是由于哪个锁引起的，通常是因为MySQL或InnoDB内核级的锁引起的建议：如果耗时较大再关注即可，一般情况下都还好  |
| Sending data     | 从server端发送数据到客户端，也有可能是接收存储引擎层返回的数据，再发送给客户端，数据量很大时尤其经常能看见，备注：Sending Data不是网络发送，是从硬盘读取，发送到网络是Writing to net。建议：通过索引或加上LIMIT，减少需要扫描并且发送给客户端的数据量    |
| Sorting result     | 正在对结果进行排序，类似Creating sort index，不过是正常表，而不是在内存表中进行排序建议：创建适当的索引     |
| Table lock     | 表级锁，没什么好说的，要么是因为MyISAM引擎表级锁，要么是其他情况显式锁表     |
| create sort index     | 当前的SELECT中需要用到临时表在进行ORDER BY排序。建议：创建适当的索引     |


## 四、mysql性能调优

### 1、减少不必要的商业需求
例如：实时数据

### 2、系统架构设计的影响
造成应用层设计不合理的主要原因还是面向对象的思维太过深入，以及为了减少自己代码的开发逻辑和对程序接口的过度依赖。
(1) 不适合在数据库中存放的数据：
* 二进制多媒体数据
* 流水队列数据（经常insert，update和delete）
* 超大文本数据

(2) 合理利用应用层的cache
* 访问频繁但很少更改的数据适合用cache

(3) 精简数据层的实现
* 减少对循环嵌套的依赖，合理利用sql语句
* 减少对sql语句的过度依赖（减少IO操作）
* 减少sql语句的重复执行

### 3、Query语句的调优
#### (1) sql语句在Mysql中执行的大致过程：
当Mysql Server的连接线程接收到Client端发送过来的sql请求后，会经过一系列的分解Parse，进行相应的分析。然后，Mysql会通过查询优化器模块（Optimizer）根据该sql所涉及的数据表的相关统计信息进行计算分析，然后在得出一个Mysql认为最合理最优化的数据访问方式，也就是我们常说的执行计划，然后再根据所得到的执行计划通过调用存储引擎接口来获取相应数据。然后再将存储引擎返回的数据进行相关处理，并以Client端所要求的格式作为结果集返回给Client端的应用程序。

#### (2) 基本优化思路
* 优化更需要优化的Query（高并发低消耗的查询）
* 定位优化对象的性能瓶颈（IO，CPU消耗分别对应数据访问，数据运算）
* 明确的优化目标
* 从explain入手（了解执行计划）
#### (3) 基本原则
* 多使用profile
* 永远用小结果集驱动大的结果集
* 尽可能在索引中完成排序
* 只取出自己需要的cloumns
* 仅仅使用最有效的过滤手段
* 尽可能避免复杂的join和子查询

#### (4) 通过开启慢查日志发现有问题的sql语句
* 开启慢查日志 set global slow_query_log=on;
* 设置日志存储路径 set global slow_query_log_file='path';
* 设置记录不用到索引的查询 set global log_queries_not_using_indexes=on;
* 设置超过时间的查询 set global long_query_time=1;
* 通过自带的mysqldumpslow分析日志 
* 通过功能更加丰富的pt-query-digest分析日志
* 查询次数多且每次查询占用时间长的SQL,通常为pt-query-digest分析的前几个查询
* IO大的SQL,注意pt-query-digest分析中的Rows examine项
* 未命中索引的SQL,注意pt-query-digest分析中Rows examine和Rows send的对比
* pt-query-digest 日志文件
#### (5) 内置函数的优化
像max(),min()等需要排序的函数使用索引进行优化
count(*) == count(value or null)

#### (6) join语句的优化
Mysql只使用一种连接算法：Nested Loop Join 
* a、尽可能减少join语句中nested loop的循环总次数，即用小结果集驱动大结果集。 
* b、优先优化循环的内层操作。 
* c、保证join语句中被驱动表上join条件字段已经被索引。 
* d、当无法保证第三点时，不要吝惜join buffer的设置。

#### (7) order by的优化
① 在排序字段上建立有序索引。
② 当无法满足第一点时，会使用排序算法
Extra: Using filesort 
当explain中只出现Using filesort时表示使用了第一种算法。该算法在取得第一个表之后，先根据排序条件将该字段与行指针放入Sort Buffer中进行排序。然后再利用排序后的数据根据行指针返回第一个表的数据，作为驱动结果集来连接到第二个表。可以看出多了一步IO操作去连接回第一张表。

Extra: Using temporary; Using filesort 
当我们的排序操作在join连接之后时，就会出现使用Using temporary; Using filesort，Mysql会先对表进行连接操作，然后将结果集放入临时表，再进行filesort，最后得到有序的结果集。显而意见，第二种算法更加高效，用空间换取时间，减少了IO操作。

③ 使用配置
* 加大max_length_for_sort_data当我们返回字段的最大长度小于这个参数时，Mysql就会选择第二种算法。
* 加大sort_buffer_size让Mysql在排序过程中对需要排序的数据进行分段。

#### (8) group by的优化
**有索引的情况下**
使用松散（loose）索引扫描实现group by，即Mysql完全利用索引扫描来实现group by 
Extra: Using where, Using index for group-by 
需要的条件： 
group by条件字段必须在同一个索引中最前面的连续位置。
在适用group by的同时，只能适用max()和min()两个聚集函数。
如果引用到了该索引中group by条件之外的字段条件时，必须以常量形式存在（不能是范围）。
使用紧凑（tight）索引扫描实现group，当group by不是以索引中的第一个开始，但是有条件where以常量形式引用到了索引中的第一个。通过该常量来引用缺失的索引键。 
Extra: Using where, Using index

**没有索引的情况下**
使用临时表 
Extra: Using where, Using index, Using temporary, Using filesort 
从执行计划可以很明显的看出来：通过索引找到我们需要的数据，然后创建了临时表，又进行了排序操作，最终才得到我们需要的group by结果。

**优化思路**
* 尽量利用索引
* 提供足够大的sort_buffer_size

#### (9) 合理利用索引
**索引的类型**
* B-Tree索引：在innodb中通过主键来访问数据效率是非常高的。 
* Hash索引：
优点：能通过hash算法得到存储位置，效率很高。
缺点：由于自身限制同时存在很多弊端。 
不能满足范围查询（因为hash算法转换后的值的大小与原值不对应）
不同用来避免数据的排序
不同利用部分索引键查询
由于冲突，必须每次都对表扫描
当有大量hash值相等时效率低
 
* FullText索引：主要用来代替效率低下的like'%%'操作。
 
* R-Tree索引：主要用来解决空间数据检索的问题。在Mysql中，支持一种用来存放空间信息的数据类型GEOMETRY。

**索引的利弊**
* 提高检索速率，降低IO成本。
* 降低数据排序成本（B-Tree的有序性）
* 提高更新操作的IO成本。
* 浪费空间。

**什么时候应该用索引**
* 较频繁的查询字段
* 唯一性较强的字段（太差的话就算频繁查询也不适合）
* 更新频繁的字段不适合做索引

**单键索引还是组合索引**
* 做到尽量让一个索引被多个query语句所利用,尽量减少同一个表上索引的数量。

**索引相关sql语句**
创建索引：
create index 索引名 on 表名(字段名列表);
强制使用索引：
select * from 表 force index(索引名) where ...;

**索引优化**
* 如何选择组合索引：离散程度大的列在前
* 主键已经默认是唯一索引了，所以primay key的主键不用再设置unique唯一索引了
* 冗余索引，是指多个索引的前缀列相同，或者在联合索引中包含了主键的索引，因为innodb会在每个索引后面自动加上主键。如果我们创建了(area, age, salary)的复合索引，那么其实相当于创建了(area,age,salary)、(area,age)、(area)三个索引，这被称为最佳左前缀特性。因此我们在创建复合索引时应该将最常用作限制条件的列放在最左边，依次递减。
* 不使用NOT IN和<>操作,NOT IN和<>操作都不会使用索引将进行全表扫描。NOT IN可以NOT EXISTS代替，id<>3则可使用id>3 or id<3来代替。
* mysql查询只使用一个索引，因此如果where子句中已经使用了索引的话，那么order by中的列是不会使用索引的。因此数据库默认排序可以符合要求的情况下不要使用排序操作；尽量不要包含多个列的排序，如果需要最好给这些列创建复合索引。
* 查询重复，冗余索引的工具pt-duplicate-key-checker
* pt-duplicate-key-checker -u root -p '密码' -h ip地址

#### （10）特定类型查询优化
**优化COUNT()查询**
COUNT() 可能是被大家误解最多的函数了，它有两种不同的作用，其一是统计某个列值的数量，其二是统计行数。
统计列值时，要求列值是非空的，它不会统计 NULL。如果确认括号中的表达式不可能为空时，实际上就是在统计行数。
最简单的就是当使用 COUNT(*) 时，并不是我们所想象的那样扩展成所有的列，实际上，它会忽略所有的列而直接统计行数。
我们最常见的误解也就在这儿，在括号内指定了一列却希望统计结果是行数，而且还常常误以为前者的性能会更好。
但实际并非这样，如果要统计行数，直接使用 COUNT(*)，意义清晰，且性能更好。
有时候某些业务场景并不需要完全精确的 COUNT 值，可以用近似值来代替，EXPLAIN 出来的行数就是一个不错的近似值，而且执行 EXPLAIN 并不需要真正地去执行查询，所以成本非常低。
通常来说，执行 COUNT() 都需要扫描大量的行才能获取到精确的数据，因此很难优化，MySQL 层面还能做得也就只有覆盖索引了。
如果还不能解决问题，只有从架构层面解决了，比如添加汇总表，或者使用 Redis 这样的外部缓存系统。

**优化关联查询**
在大数据场景下，表与表之间通过一个冗余字段来关联，要比直接使用JOIN有更好的性能。

如果确实需要使用关联查询的情况下，需要特别注意的是：
* 确保ON和USING字句中的列上有索引。在创建索引的时候就要考虑到关联的顺序。
* 当表A和表B用列c关联的时候，如果优化器关联的顺序是A、B，那么就不需要在A表的对应列上创建索引。
* 没有用到的索引会带来额外的负担，一般来说，除非有其他理由，只需要在关联顺序中的第二张表的相应列上创建索引（具体原因下文分析）。
* 确保任何的GROUP BY和ORDER BY中的表达式只涉及到一个表中的列，这样MySQL才有可能使用索引来优化。

要理解优化关联查询的第一个技巧，就需要理解MySQL是如何执行关联查询的。
当前MySQL关联执行的策略非常简单，它对任何的关联都执行嵌套循环关联操作，即先在一个表中循环取出单条数据，然后在嵌套循环到下一个表中寻找匹配的行，依次下去，直到找到所有表中匹配的行为为止。然后根据各个表匹配的行，返回查询中需要的各个列。

太抽象了？以上面的示例来说明，比如有这样的一个查询：
```
SELECT A.xx,B.yy 
FROM A INNER JOIN B USING(c)
WHERE A.xx IN (5,6)
```
假设 MySQL 按照查询中的关联顺序 A、B 来进行关联操作，那么可以用下面的伪代码表示 MySQL 如何完成这个查询：
```
outer_iterator = SELECT A.xx,A.c FROM A WHERE A.xx IN (5,6);
outer_row = outer_iterator.next;
while(outer_row) {
   inner_iterator = SELECT B.yy FROM B WHERE B.c = outer_row.c;
   inner_row = inner_iterator.next;
   while(inner_row) {
       output[inner_row.yy,outer_row.xx];
       inner_row = inner_iterator.next;
   }
   outer_row = outer_iterator.next;
}
```
可以看到，最外层的查询是根据 A.xx 列来查询的，A.c 上如果有索引的话，整个关联查询也不会使用。
再看内层的查询，很明显 B.c 上如果有索引的话，能够加速查询，因此只需要在关联顺序中的第二张表的相应列上创建索引即可。

**优化LIMIT分页**
当需要分页操作时，通常会使用 LIMIT 加上偏移量的办法实现，同时加上合适的 ORDER BY 字句。
如果有对应的索引，通常效率会不错，否则，MySQL 需要做大量的文件排序操作。
一个常见的问题是当偏移量非常大的时候，比如：LIMIT 10000 20 这样的查询，MySQL 需要查询 10020 条记录然后只返回 20 条记录，前面的 10000 条都将被抛弃，这样的代价非常高。

优化这种查询一个最简单的办法就是尽可能的使用覆盖索引扫描，而不是查询所有的列。

然后根据需要做一次关联查询再返回所有的列。对于偏移量很大时，这样做的效率会提升非常大。考虑下面的查询：

```
SELECT film_id,description FROM film ORDER BY title LIMIT 50,5;
```
如果这张表非常大，那么这个查询最好改成下面的样子：

```
SELECT film.film_id,film.description
FROM film INNER JOIN (
   SELECT film_id FROM film ORDER BY title LIMIT 50,5
) AS tmp USING(film_id);
```
这里的延迟关联将大大提升查询效率，让 MySQL 扫描尽可能少的页面，获取需要访问的记录后在根据关联列回原表查询所需要的列。
有时候如果可以使用书签记录上次取数据的位置，那么下次就可以直接从该书签记录的位置开始扫描，这样就可以避免使用 OFFSET，比如下面的查询：

```
SELECT id FROM t LIMIT 10000, 10;
改为：
SELECT id FROM t WHERE id > 10000 LIMIT 10;
```
其他优化的办法还包括使用预先计算的汇总表，或者关联到一个冗余表，冗余表中只包含主键列和需要做排序的列。

**优化UNION**
MySQL 处理 UNION 的策略是先创建临时表，然后再把各个查询结果插入到临时表中，最后再来做查询。因此很多优化策略在 UNION 查询中都没有办法很好的时候，经常需要手动将 WHERE、LIMIT、ORDER BY 等字句“下推”到各个子查询中，以便优化器可以充分利用这些条件先优化。
除非确实需要服务器去重，否则就一定要使用UNION ALL，如果没有ALL关键字，MySQL会给临时表加上DISTINCT选项，这会导致整个临时表的数据做唯一性检查，这样做的代价非常高。
当然即使使用ALL关键字，MySQL总是将结果放入临时表，然后再读出，再返回给客户端。
虽然很多时候没有这个必要，比如有时候可以直接把每个子查询的结果返回给客户端。

### 4、数据库结构优化
#### (1) 选择合适的数据类型
* 使用可存下数据的最小的数据类型
* 使用简单地数据类型，Int在mysql的处理比varchar简单
* 尽可能使用not null定义字段
* 尽量少用text，非用不可最好分表

① 用int来存储日期时间以节省空间：
* 插入时使用UNIX_TIMESTAMP()将datetime转成int
* 查询时使用FROM_UNIXTIME()将int转成datetime

② 用bigint来存储ip地址：
* 插入时使用INET_ATON()来将ip地址转成bigint
* 查询时使用INET_NTOA()来将bigint转成ip地址

#### (2) 表的范式化
优化到第三范式以上。

#### (3) 反范式化
为了优化查询效率，以空间换时间。

#### (4) 表的垂直拆分
① 把原来有很多列的表拆分成多个表，原则是：
* 把不常用的字段单独存放到一个表中
* 把大字段独立存放在一个表中
* 把经常使用的字段放在一起

#### (5) 表的水平拆分
① 为了解决单表数据量过大的问题，每个水平拆分表的结构完全一致
方法: 
a、对id进行hash运算，可以取mod 
挑战: 
b、跨分区进行数据查询 
c、统计及后台报表操作

### 5、系统配置优化
#### (1) 操作系统的优化
网络方面，修改/etc/sysctl.conf文件，增加tcp支持的队列数，减少断开连接时，资源的回收。
打开文件数的限制。修改/etc/security/limits.conf文件，增加一下内容以修改打开文件数量的限制。
关闭iptables，selinux等防火墙软件。使用硬件防火墙。
#### (2) Mysql配置文件
Mysql可以通过启动时指定参数和使用配置文件两种方法进行配置。大多数情况下默认的配置文件位于/etc/my.cnf或/etc/mysql/my.cnf，查找配置文件的顺序可以用该命令获取：
usr/sbin/mysqld --verbose --help | grep -A 1 'Defaulto options'
① 重要参数：
innodb_buffer_pool_size配置innodb的缓冲池，如果数据库中只有innodb表，则推荐配置为总内存的75%。
innodb_flush_log_at_trx_commit(0|1|2)决定数据库事务刷新到磁盘的时间，0是每秒刷新一次。1是每次提交刷新一次，安全性最高。2是先提交到缓冲区，再每秒刷新一次，效率最高。

innodb_file_per_table控制innodb每一个表使用独立的表空间，默认是OFF，造成IO瓶颈。推荐设置ON。
② 常用参数：
innodb_buffer_pool_instances配置缓冲池的个数，默认是一个。
innodb_log_buffer_size配置日志缓冲区的大小，一般不用太大。
innodb_read_io_threads(默认是4) 
innodb_write_io_threads(默认是4)决定innodb读写的IO进程数。
innodb_stats_on_metadata配置mysql在什么情况下刷新innodb表的统计信息。



## 五、mysql的binlog
binlog有三种形式Statement、Row、Mixedlevel。
### Statement
Statement：每一条会修改数据的sql都会记录在binlog中。
优点：不需要记录每一行的变化，减少了binlog日志量，节约了IO，提高性能。(相比row能节约多少性能 与日志量，这个取决于应用的SQL情况，正常同一条记录修改或者插入row格式所产生的日志量还小于Statement产生的日志量，但是考虑到如果带条 件的update操作，以及整表删除，alter表等操作，ROW格式会产生大量日志，因此在考虑是否使用ROW格式日志时应该跟据应用的实际情况，其所 产生的日志量会增加多少，以及带来的IO性能问题。)
缺点：由于记录的只是执行语句，为了这些语句能在slave上正确运行，因此还必须记录每条语句在执行的时候的 一些相关信息，以保证所有语句能在slave得到和在master端执行时候相同 的结果。另外mysql 的复制,像一些特定函数功能，slave可与master上要保持一致会有很多相关问题(如sleep()函数， last_insert_id()，以及user-defined functions(udf)会出现问题).
使用以下函数的语句也无法被复制：
* LOAD_FILE()
* UUID()
* USER()
* FOUND_ROWS()
* SYSDATE() (除非启动时启用了 --sysdate-is-now 选项)同时在INSERT ...SELECT 会产生比 RBR 更多的行级锁

### Row
Row:不记录sql语句上下文相关信息，仅保存哪条记录被修改。
优点： binlog中可以不记录执行的sql语句的上下文相关的信息，仅需要记录那一条记录被修改成什么了。所以rowlevel的日志内容会非常清楚的记录下 每一行数据修改的细节。而且不会出现某些特定情况下的存储过程，或function，以及trigger的调用和触发无法被正确复制的问题
缺点:所有的执行的语句当记录到日志中的时候，都将以每行记录的修改来记录，这样可能会产生大量的日志内容,比 如一条update语句，修改多条记录，则binlog中每一条修改都会有记录，这样造成binlog日志量会很大，特别是当执行alter table之类的语句的时候，由于表结构修改，每条记录都发生改变，那么该表每一条记录都会记录到日志中。

### Mixedlevel
Mixedlevel: 是以上两种level的混合使用，一般的语句修改使用statment格式保存binlog，如一些函数，statement无法完成主从复制的操作，则 采用row格式保存binlog,MySQL会根据执行的每一条具体的sql语句来区分对待记录的日志形式，也就是在Statement和Row之间选择 一种.新版本的MySQL中队row level模式也被做了优化，并不是所有的修改都会以row level来记录，像遇到表结构变更的时候就会以statement模式来记录。至于update或者delete等修改数据的语句，还是会记录所有行的变更。

## 六、事务及ACID原理
### 1、什么是事务？
事务(Transaction)，一般是指要做的或所做的事情。在计算机术语中是指访问并可能更新数据库中各种数据项的一个程序执行单元(unit)。事务由事务开始(begin transaction)和事务结束(end transaction)之间执行的全体操作组成。在关系数据库中，一个事务可以是一组SQL语句或整个程序。

### 2、为什么要有事务
一个数据库事务通常包含对数据库进行读或写的一个操作序列。它的存在包含有以下两个目的：
* 为数据库操作提供了一个从失败中恢复到正常状态的方法，同时提供了数据库在异常状态下仍能保持一致性的方法。
* 当多个应用程序在并发访问数据库时，可以在这些应用程序之间提供一个隔离方法，保证彼此的操作互相干扰。

### 3、事务特性及原理
事务具有4个特性：原子性、一致性、隔离性、持久性。这四个属性通常称为 ACID 特性。
#### (1)原子性
**原子性（atomicity）**：一个事务应该是一个不可分割的工作单位，事务中包括的操作要么都成功，要么都不成功。
**实现原理**：undo log
    在说明原子性原理之前，首先介绍一下 MySQL 的事务日志。MySQL 的日志有很多种，如二进制日志、错误日志、查询日志、慢查询日志等。此外 InnoDB 存储引擎还提供了两种事务日志：
redo log(重做日志)
undo log(回滚日志)
其中 redo log 用于保证事务持久性；undo log 则是事务原子性和隔离性实现的基础。
下面说回 undo log。实现原子性的关键，是当事务回滚时能够撤销所有已经成功执行的sql语句。

InnoDB实现回滚，靠的是undo log：
当事务对数据库进行修改时，InnoDB 会生成对应的 undo log。
如果事务执行失败或调用了 rollback，导致事务需要回滚，便可以利用 undo log 中的信息将数据回滚到修改之前的样子。
undo log 属于逻辑日志，它记录的是 sql 执行相关的信息。当发生回滚时，InnoDB 会根据 undo log 的内容做与之前相反的工作：
* 对于每个 insert，回滚时会执行 delete。
* 对于每个 delete，回滚时会执行 insert。
* 对于每个 update，回滚时会执行一个相反的 update，把数据改回去。

以 update 操作为例：当事务执行 update 时，其生成的 undo log 中会包含被修改行的主键(以便知道修改了哪些行)、修改了哪些列、这些列在修改前后的值等信息，回滚时便可以使用这些信息将数据还原到update之前的状态。

#### (2)持久性
**持久性（durability）**：一个事务一旦成功提交，它对数据库中数据的改变就应该是永久性的。接下来的其他操作或故障不应该对其有任何影响。
事物之间的几个特性并不是一组同等的概念：
如果在任何时刻都只有一个事物，那么其天然是具有隔离性的，这时只要保证原子性就能具有一致性。
如果存在并发的情况下，就需要保证原子性和隔离性才能保证一致性。
**实现原理：redo log**
    redo log 和 undo log 都属于 InnoDB 的事务日志。下面先聊一下 redo log 存在的背景。
InnoDB 作为 MySQL 的存储引擎，数据是存放在磁盘中的，但如果每次读写数据都需要磁盘 IO，效率会很低。为此，InnoDB 提供了缓存(Buffer Pool)，Buffer Pool 中包含了磁盘中部分数据页的映射，作为访问数据库的缓冲：当从数据库读取数据时，会首先从 Buffer Pool 中读取，如果 Buffer Pool 中没有，则从磁盘读取后放入 Buffer Pool。当向数据库写入数据时，会首先写入 Buffer Pool，Buffer Pool 中修改的数据会定期刷新到磁盘中（这一过程称为刷脏）。
   Buffer Pool 的使用大大提高了读写数据的效率，但是也带来了新的问题：如果 MySQL 宕机，而此时 Buffer Pool 中修改的数据还没有刷新到磁盘，就会导致数据的丢失，事务的持久性无法保证。于是，redo log 被引入来解决这个问题：当数据修改时，除了修改 Buffer Pool 中的数据，还会在 redo log 记录这次操作；当事务提交时，会调用 fsync 接口对 redo log 进行刷盘。
    如果MySQL宕机，重启时可以读取redo log中的数据，对数据库进行恢复。redo log采用的是WAL（Write-ahead logging，预写式日志），所有修改先写入日志，再更新到 Buffer Pool，保证了数据不会因 MySQL 宕机而丢失，从而满足了持久性要求。既然redo log也需要在事务提交时将日志写入磁盘，为什么它比直接将Buffer Pool中修改的数据写入磁盘(即刷脏)要快呢？

主要有以下两方面的原因：
* 刷脏是随机IO，因为每次修改的数据位置随机，但写redo log是追加操作，属于顺序IO。
* 刷脏是以数据页（Page）为单位的，MySQL默认页大小是 16KB，一个Page上一个小修改都要整页写入；而redo log中只包含真正需要写入的部分，无效IO大大减少。



**redo log 与 binlog**
我们知道，在 MySQL 中还存在 binlog(二进制日志)也可以记录写操作并用于数据的恢复，但二者是有着根本的不同的。
* 作用不同：
redo log 是用于 crash recovery 的，保证 MySQL 宕机也不会影响持久性；
binlog 是用于 point-in-time recovery 的，保证服务器可以基于时间点恢复数据，此外 binlog 还用于主从复制。

* 层次不同：
redo log 是 InnoDB 存储引擎实现的，
而 binlog 是 MySQL 的服务器层(可以参考文章前面对 MySQL 逻辑架构的介绍)实现的，同时支持 InnoDB 和其他存储引擎。

* 内容不同：
redo log 是物理日志，内容基于磁盘的 Page。
binlog 是逻辑日志，内容是一条条 sql。

* 写入时机不同：
redo log 的写入时机相对多元。前面曾提到，当事务提交时会调用 fsync 对 redo log 进行刷盘；这是默认情况下的策略，修改 innodb_flush_log_at_trx_commit 参数可以改变该策略，但事务的持久性将无法保证。除了事务提交时，还有其他刷盘时机：如 master thread 每秒刷盘一次 redo log 等，这样的好处是不一定要等到 commit 时刷盘，commit 速度大大加快。binlog在事务提交时写入。

#### (3)隔离性
**隔离性（isolation）**：一个事务的执行不能被其他事务干扰。即一个事务内部的操作及使用的数据在事物未提交前对并发的其他事务是隔离的，并发执行的各个事务之间不能互相影响。

隔离性的原理，主要可以分为两个方面：
* (一个事务)写操作对(另一个事务)写操作的影响：锁机制保证隔离性。
* (一个事务)写操作对(另一个事务)读操作的影响：MVCC 保证隔离性。

**锁机制**

首先来看两个事务的写操作之间的相互影响。隔离性要求同一时刻只能有一个事务对数据进行写操作，InnoDB通过锁机制来保证这一点。
锁机制的基本原理可以概括为：
* 事务在修改数据之前，需要先获得相应的锁。
* 获得锁之后，事务便可以修改数据。
* 该事务操作期间，这部分数据是锁定的，其他事务如果需要修改数据，需要等待当前事务提交或回滚后释放锁。
行锁与表锁：按照粒度，锁可以分为表锁、行锁以及其他位于二者之间的锁。

表锁在操作数据时会锁定整张表，并发性能较差；行锁则只锁定需要操作的数据，并发性能好。但是由于加锁本身需要消耗资源(获得锁、检查锁、释放锁等都需要消耗资源)，因此在锁定数据较多情况下使用表锁可以节省大量资源。

共享锁和排他锁：
* 排它锁（Exclusive），又称为X锁，写锁。
* 共享锁（Shared），又称为S 锁，读锁。

读写锁之间有以下的关系：
* 一个事务对数据对象O加了 S 锁，可以对 O进行读取操作，但是不能进行更新操作。加锁期间其它事务能对O 加 S 锁，但是不能加 X 锁。
* 一个事务对数据对象 O 加了 X 锁，就可以对 O 进行读取和更新。加锁期间其它事务不能对 O 加任何锁。即读写锁之间的关系可以概括为：多读单写

**MVCC**
多版本并发控制(Multi-Version Concurrency Control, MVCC)是MySQL中基于乐观锁理论实现隔离级别的方式，用于实现读已提交和可重复读取隔离级别的实现。

实现(隔离级别为可重复读)
MVCC中的重要概念：
* **系统版本号**：一个递增的数字，每开始一个新的事务，系统版本号就会自动递增。
* **事务版本号**：事务开始时的系统版本号。
* **创建版本号**：创建一行数据时，将当前系统版本号作为创建版本号赋值
* **删除版本号**：删除一行数据时，将当前系统版本号作为删除版本号赋值

MVCC如何实现CRUD：
* **SELECT**
select时读取数据的规则为：创建版本号<=当前事务版本号，删除版本号为空或>当前事务版本号。创建版本号<=当前事务版本号保证取出的数据不会有后启动的事物中创建的数据。这也是为什么在开始的示例中我们不会查出后来添加的数据的原因；删除版本号为空或>当前事务版本号保证了至少在该事物开启之前数据没有被删除，是应该被查出来的数据。
* **INSERT**
insert时将当前的系统版本号赋值给创建版本号字段。
* **UPDATE**
 插入一条新纪录，保存当前事务版本号为行创建版本号，同时保存当前事务版本号到原来删除的行，实际上这里的更新是通过delete和insert实现的。
* **DELETE**
删除时将当前的系统版本号赋值给删除版本号字段，标识该行数据在那一个事物中会被删除，即使实际上在位commit时该数据没有被删除。根据select的规则后开启懂数据也不会查询到该数据。


#### (4)一致性
**一致性（consistency）**：事务必须是使数据库从一个一致性状态变到另一个一致性状态。一致性与原子性是密切相关的。
可以说，一致性是事务追求的最终目标：前面提到的原子性、持久性和隔离性，都是为了保证数据库状态的一致性。此外，除了数据库层面的保障，一致性的实现也需要应用层面进行保障。


实现一致性的措施包括：
* 保证原子性、持久性和隔离性，如果这些特性无法保证，事务的一致性也无法保证。
* 数据库本身提供保障，例如不允许向整形列插入字符串值、字符串长度不能超过列的限制等。 
* 应用层面进行保障，例如如果转账操作只扣除转账者的余额，而没有增加接收者的余额，无论数据库实现的多么完美，也无法保证状态的一致。


### 4、数据库并发事务中存在的问题
如果不考虑事务的隔离性，会发生以下几种问题：
* **脏读**：脏读是指在一个事务处理过程里读取了另一个未提交的事务中的数据。当一个事务正在多次修改某个数据，而在这个事务中这多次的修改都还未提交，这时一个并发的事务来访问该数据，就会造成两个事务得到的数据不一致。具体示例如下：
 ![](https://i.imgur.com/00vdmiw.png)


* **不可重复读**：不可重复读是指在对于数据库中的某条数据，一个事务范围内多次查询返回不同的数据值(这里不同是指某一条或多条数据的内容前后不一致，但数据条数相同)，这是由于在查询间隔，该事物需要用到的数据被另一个事务修改并提交了。不可重复读和脏读的区别是，脏读是某一事务读取了另一个事务未提交的脏数据，而不可重复读则是读取了其他事务提交的数据。需要注意的是在某些情况下不可重复读并不是问题。具体示例如下：
![](https://i.imgur.com/16JuiLP.png)


* **幻读**：幻读是事务非独立执行时发生的一种现象。例如事务T1对一个表中所有的行的某个数据项做了从“1”修改为“2”的操作，这时事务T2又对这个表中插入了一行数据项，而这个数据项的数值还是为“1”并且提交给数据库。而操作事务T1的用户如果再查看刚刚修改的数据，会发现还有一行没有修改，其实这行是从事务T2中添加的，就好像产生幻觉一样，这就是发生了幻读。幻读和不可重复读都是读取了另一条已经提交的事务(这点就脏读不同)，所不同的是不可重复读可能发生在update,delete操作中，而幻读发生在insert操作中。具体示例如下：
![](https://i.imgur.com/5LqNa9d.png)

### 5、事务的隔离级别
在事物中存在以下几种隔离级别：
* **读未提交(Read Uncommitted)**：解决更新丢失问题。如果一个事务已经开始写操作，那么其他事务则不允许同时进行写操作，但允许其他事务读此行数据。该隔离级别可以通过“排他写锁”实现，即事物需要对某些数据进行修改必须对这些数据加 X 锁，读数据不需要加 S 锁。
 
* **读已提交(Read Committed)**：解决了脏读问题。读取数据的事务允许其他事务继续访问该行数据，但是未提交的写事务将会禁止其他事务访问该行。这可以通过“瞬间共享读锁”和“排他写锁”实现， 即事物需要对某些数据进行修改必须对这些数据加 X 锁，读数据时需要加上 S 锁，当数据读取完成后立刻释放 S 锁，不用等到事物结束。
 
* **可重复读取(Repeatable Read)**：禁止不可重复读取和脏读取，但是有时可能出现幻读数据。读取数据的事务将会禁止写事务(但允许读事务)，写事务则禁止任何其他事务。Mysql默认使用该隔离级别。这可以通过“共享读锁”和“排他写锁”实现，即事物需要对某些数据进行修改必须对这些数据加 X 锁，读数据时需要加上 S 锁，当数据读取完成并不立刻释放 S 锁，而是等到事物结束后再释放。
 
* **串行化(Serializable)**：解决了幻读的问题的。提供严格的事务隔离。它要求事务序列化执行，事务只能一个接着一个地执行，不能并发执行。仅仅通过“行级锁”是无法实现事务序列化的，必须通过其他机制保证新插入的数据不会被刚执行查询操作的事务访问到。

### 6、隔离级别与读问题的关系
![](https://i.imgur.com/fgC0Iop.png)
    在实际应用中，读未提交在并发时会导致很多问题，而性能相对于其他隔离级别提高却很有限，因此使用较少。可串行化强制事务串行，并发效率很低，只有当对数据一致性要求极高且可以接受没有并发时使用，因此使用也较少。因此在大多数数据库系统中，默认的隔离级别是读已提交(如 Oracle)或可重复读（后文简称 RR）。


### 7、ACID特性及其实现原理小结

* 原子性：语句要么全执行，要么全不执行，是事务最核心的特性。事务本身就是以原子性来定义的；实现主要基于 undo log。
* 持久性：保证事务提交后不会因为宕机等原因导致数据丢失；实现主要基于 redo log。
* 隔离性：保证事务执行尽可能不受其他事务影响；InnoDB 默认的隔离级别是 RR，RR 的实现主要基于锁机制、数据的隐藏列、undo log 和类next-key lock机制。
* 一致性：事务追求的最终目标，一致性的实现既需要数据库层面的保障，也需要应用层面的保障。



## 附：
### 一、mysql常见问题
1、MySQL的复制原理以及流程
* 基本原理流程，3个线程以及之间的关联；
主：binlog线程——记录下所有改变了数据库数据的语句，放进master上的binlog中；
从：io线程——在使用start slave 之后，负责从master上拉取 binlog 内容，放进 自己的relay log中；
从：sql执行线程——执行relay log中的语句；


2、MySQL中myisam与innodb的区别，至少5点
* (1)、问5点不同；
1>.InnoDB支持事物，而MyISAM不支持事物
2>.InnoDB支持行级锁，而MyISAM支持表级锁
3>.InnoDB支持MVCC, 而MyISAM不支持
4>.InnoDB支持外键，而MyISAM不支持
5>.InnoDB不支持全文索引，而MyISAM支持。

* (2)、innodb引擎的4大特性
插入缓冲（insert buffer),二次写(double write),自适应哈希索引(ahi),预读(read ahead)

* (3)、2者selectcount(*)哪个更快，为什么
myisam更快，因为myisam内部维护了一个计数器，可以直接调取。


3、MySQL中varchar与char的区别以及varchar(50)中的50代表的涵义
* (1)、varchar与char的区别
 char是一种固定长度的类型，varchar则是一种可变长度的类型
* (2)、varchar(50)中50的涵义
最多存放50个字符，varchar(50)和(200)存储hello所占空间一样，但后者在排序时会消耗更多内存，因为order by col采用fixed_length计算col长度(memory引擎也一样)
* (3)、int（20）中20的涵义
是指显示字符的长度
但要加参数的，最大为255，比如它是记录行数的id,插入10笔资料，它就显示00000000001 00000000010，当字符的位数超过11,它也只显示11位，如果你没有加那个让它未满11位就前面加0的参数，它不会在前面加0
20表示最大显示宽度为20，但仍占4字节存储，存储范围不变；
* (4)、mysql为什么这么设计
对大多数应用没有意义，只是规定一些工具用来显示字符的个数；int(1)和int(20)存储和计算均一样；

4、问了下MySQL数据库cpu飙升到500%的话他怎么处理？
列出所有进程  show processlist  观察所有进程  多秒没有状态变化的(干掉)
查看超时日志或者错误日志 (做了几年开发,一般会是查询以及大批量的插入会导致cpu与i/o上涨,,,,当然不排除网络状态突然断了,,导致一个请求服务器只接受到一半，比如where子句或分页子句没有发送,,当然的一次被坑经历)

5、备份计划，mysqldump以及xtranbackup的实现原理
* (1)、备份计划；
这里每个公司都不一样，您别说那种1小时1全备什么的就行
* (2)、备份恢复时间；
这里跟机器，尤其是硬盘的速率有关系，以下列举几个仅供参考
20G的2分钟（mysqldump）
80G的30分钟(mysqldump)
111G的30分钟（mysqldump)
288G的3小时（xtra)
3T的4小时（xtra)
逻辑导入时间一般是备份时间的5倍以上

* (3)、xtrabackup实现原理
在InnoDB内部会维护一个redo日志文件，我们也可以叫做事务日志文件。事务日志会存储每一个InnoDB表数据的记录修改。当InnoDB启动时，InnoDB会检查数据文件和事务日志，并执行两个步骤：它应用（前滚）已经提交的事务日志到数据文件，并将修改过但没有提交的数据进行回滚操作。

6、mysqldump中备份出来的sql，如果我想sql文件中，一行只有一个insert....value()的话，怎么办？如果备份需要带上master的复制点信息怎么办？
--skip-extended-insert
[root@helei-zhuanshu ~]# mysqldump -uroot -p helei --skip-extended-insert
Enter password:
  KEY `idx_c1` (`c1`),
  KEY `idx_c2` (`c2`)
) ENGINE=InnoDB AUTO_INCREMENT=51 DEFAULT CHARSET=latin1;
/*!40101 SET character_set_client = @saved_cs_client */;


 --Dumping data for table `helei`


LOCK TABLES `helei` WRITE;
/*!40000 ALTER TABLE `helei` DISABLE KEYS */;
INSERT INTO `helei` VALUES (1,32,37,38,'2016-10-18 06:19:24','susususususususususususu');
INSERT INTO `helei` VALUES (2,37,46,21,'2016-10-18 06:19:24','susususususu');
INSERT INTO `helei` VALUES (3,21,5,14,'2016-10-18 06:19:24','susu');


7、innodb的读写参数优化
* (1)、读取参数
global buffer pool以及 local buffer；
* (2)、写入参数；
innodb_flush_log_at_trx_commit
innodb_buffer_pool_size
* (3)、与IO相关的参数；
innodb_write_io_threads = 8
innodb_read_io_threads = 8
innodb_thread_concurrency = 0
* (4)、缓存参数以及缓存的适用场景。
query cache/query_cache_type
并不是所有表都适合使用query cache。造成query cache失效的原因主要是相应的table发生了变更
第一个：读操作多的话看看比例，简单来说，如果是用户清单表，或者说是数据比例比较固定，比如说商品列表，是可以打开的，前提是这些库比较集中，数据库中的实务比较小。
第二个：我们“行骗”的时候，比如说我们竞标的时候压测，把query cache打开，还是能收到qps激增的效果，当然前提示前端的连接池什么的都配置一样。大部分情况下如果写入的居多，访问量并不多，那么就不要打开，例如社交网站的，10%的人产生内容，其余的90%都在消费，打开还是效果很好的，但是你如果是qq消息，或者聊天，那就很要命。
第三个：小网站或者没有高并发的无所谓，高并发下，会看到 很多 qcache 锁 等待，所以一般高并发下，不建议打开query cache


8、你是如何监控你们的数据库的？你们的慢日志都是怎么查询的？
监控的工具有很多，例如zabbix，lepus，我这里用的是lepus

9、你是否做过主从一致性校验，如果有，怎么做的，如果没有，你打算怎么做？
主从一致性校验有多种工具 例如checksum、mysqldiff、pt-table-checksum等

10、你们数据库是否支持emoji表情，如果不支持，如何操作？
如果是utf8字符集的话，需要升级至utf8_mb4方可支持

11、你是如何维护数据库的数据字典的？
这个大家维护的方法都不同，我一般是直接在生产库进行注释，利用工具导出成excel方便流通。

12、你们是否有开发规范，如果有，如何执行的
有，开发规范网上有很多了，可以自己看看总结下

13、表中有大字段X(例如：text类型)，且字段X不会经常更新，以读为为主，请问
(1)、您是选择拆成子表，还是继续放一起；
(2)、写出您这样选择的理由。
答：拆带来的问题：连接消耗 + 存储拆分空间；不拆可能带来的问题：查询性能；
如果能容忍拆分带来的空间问题,拆的话最好和经常要查询的表的主键在物理结构上放置在一起(分区) 顺序IO,减少连接消耗,最后这是一个文本列再加上一个全文索引来尽量抵消连接消耗
如果能容忍不拆分带来的查询性能损失的话:上面的方案在某个极致条件下肯定会出现问题,那么不拆就是最好的选择

14、MySQL中InnoDB引擎的行锁是通过加在什么上完成(或称实现)的？为什么是这样子的？
答：InnoDB是基于索引来完成行锁 InnoDB行锁是通过给索引上的索引项加锁来实现的 只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁
例: select * from tab_with_index where id = 1 for update;
for update 可以根据条件来完成行锁锁定,并且 id 是有索引键的列,
如果 id 不是索引键那么InnoDB将完成表锁,,并发将无从谈起

15、如何从mysqldump产生的全库备份中只恢复某一个库、某一张表？
答案见：http://suifu.blog.51cto.com/9167728/1830651


16、一个6亿的表a，一个3亿的表b，通过外间tid关联，你如何最快的查询出满足条件的第50000到第50200中的这200条数据记录。
答：如果A表TID是自增长,并且是连续的,B表的ID为索引
select * from a,b where a.tid = b.id and a.tid>500000 limit 200;
如果A表的TID不是连续的,那么就需要使用覆盖索引.TID要么是主键,要么是辅助索引,B表ID也需要有索引。
select * from b , (select tid from a limit 50000,200) a where b.id = a .tid;

17、除了使用使用串行化读的隔离级别以外，如何解决幻读问题：
答：MVCC+next-key locks：next-key locks由record locks(索引加锁) 和 gap locks(间隙锁，每次锁住的不光是需要使用的数据，还会锁住这些数据附近的数据)

### 二、常见排查数据问题的方式
1、查看当前正在运行的sql语句执行时间：
select * from information_schema.PROCESSLIST where info is not null order by time desc ;

2、查看当前有那些表在使用：
show OPEN TABLES where In_use > 0;

3、查看与innodb事务先关的语句：
SELECT FROM INFORMATION_SCHEMA.INNODB_LOCKS; 
SELECT from INFORMATION_SCHEMA.INNODB_TRX;
SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS;

4、查看当前数据库的链接情况：
SELECT count(*) AS count,USER,db,SUBSTRING_INDEX(HOST, ':', 1) AS ip FROM information_schema. PROCESSLIST GROUP BY USER,db,ip ORDER BY count desc;

5、查看数据库表的碎片情况：
select round(sum(data_length/1024/1024),2) as data_length_MB, 
round(sum(index_length/1024/1024),2) as index_length_MB ,
round(sum(data_free/1024/1024),2) as data_free_MB ,table_name
from information_schema.tables where TABLE_SCHEMA= 'db_name' group by table_name order by 3 desc;

5、查看死锁，未提交事物、CHECKPIONT、BUFFER POOL、THREAD等innodb的内部信息：
show engine innodb status；

6、查看当前连接id号 
select connection_id();

7、获取锁的状态 
select * from information_schema.innodb_locks;

8、查看未提交的事务
SELECT * FROM information_schema.INNODB_TRX;

9、查看应用的数据库连接情况 
```for x in `ps aux | grep {appName} | grep admin|grep java | awk '{print $2}'`;do netstat -natp | grep {dbPort} | grep $x -c ;done```