# MySQL索引

### 1. 索引的好处

字典的目录。

Boy--> B--> Bo-->Boy-->200

A->  ... -> Boy 效率非常低，耗时非常长

索引的好处：加快查找，提高查找效率

### 2. 索引的分类

1. 单列索引：基于一列建立的索引

 1）普通索引

```mysql
没有任何限制的索引
索引类型：normal
创建索引的三种方式：
1. 建表时创建
create table table_name(
	id int,
    name varchar(50),
    sex varchar(10),
    index index_name(name)
);
2. 直接创建索引
create index index_name on table_name(column_name);
3. 修改表结构时创建索引
alter table table_name add index index_name(column_name);
```

 关于索引的一些操作：

```mysql
-- 查看数据库中所有的索引
select * from mysql.`innodb_index_stats` a where a.`database_name` = 'indextest';

-- 查看表中所有的索引
show index from student;
select * from mysql.`innodb_index_stats` a where a.`database_name` = 'indextest' and a.`table_name` like '%student%';

-- 删除索引
-- 方式1 直接删除
drop index index_age on student;

-- 方式2 修改表结构时删除
alter table student drop index index_gender;
```

2）唯一索引

```mysql
指索引的值必须唯一，允许为空。如身份证号，手机号。
索引类型：unique
创建唯一索引的三种方式
1. 创建表时创建索引
create table table_name (
	id int,
    name varchar(50),
    telephone varchar(20),
    unique index index_name(telephone)
)
2. 直接创建
create unique index index_name on student(telephone);
3. 修改表结构时创建
alter table table_name add unique index_name(telephone);
```

 3） 主键索引

```
每一张表都有一个主键。在建表时，mySQL会自动为主键创建一个主键索引，要求唯一且不为空。
```

2. 组合索引

   ```mysql
   也叫复合索引，可以创建为普通索引和唯一索引。
   -- 创建复合索引
   create index index_phone_name on student(phone_num, name);
   使用复合索引的原则：
   1. 最左前缀匹配原则
   查询的条件必须和复合索引的列的顺序保持最左匹配。
   2. SQL优化原则
   当where条件顺序与索引顺序不同，SQL会优化查询语句，以找到最符合的索引。
   ```

   

3. 全文索引

   ```mysql
   类型：fulltext
   对全文文本的检索。更像是一个搜索引擎，而不是简单地where参数匹配。
   全文索引主要用来查找文本中的关键字，而不是直接与索引中的值相比较，它更像是一个搜索引擎，基于相似度的查询，而不是简单的where语句的参数匹配。
   
   创建全文索引的三种方法：
   1. 建表时
   create table table_name(
   	...
       fulltext index_name(column_name)
   )
   2. 直接创建
   create fulltext index index_name on table_name(column_name);
   3. 修改表结构时
   alter table table_name add fulltext index_name(column_name);
   
   1. 已经有了like模糊查询，为什么还要引入全文索引？
   like + % 在文本比较少时是合适的，但是对于大量的文本数据检索，是不可想象的。全文索引在大量的数据面前，能比 like + % 快 N 倍，速度不是一个数量级，但是全文索引可能存在精度问题。
   2. MySQL5.6以前，只有MyISAM存储引擎支持全文索引；以后都支持了。
   3. 只有数据类型为char, varchar, text才可以建索引。
   4. 最小搜索长度和最大搜索长度
   MySQL 中的全文索引，有两个变量，最小搜索长度和最大搜索长度，对于长度小于最小搜索长度和大于最大搜索长度的词语，都不会被索引。通俗点就是说，想对一个词语使用全文索引搜索，那么这个词语的长度必须在以上两个变量的区间内。这两个的默认值可以使用以下命令查看:
   show variables like '%ft%';
   最大搜索长度innodb_ft_max_token_size
   最小搜索长度innodb_ft_min_token_size
   
   ```

   

4. 空间索引

```mysql 
空间索引是对空间数据类型的字段建立的索引，MYSQL中的空间数据类型有4种，分别是GEOMETRY、POINT、LINESTRING、POLYGON。
MYSQL使用SPATIAL关键字进行扩展，使得能够用于创建正规索引类型的语法创建空间索引。
创建空间索引的列，必须将其声明为NOT NULL。
create table shop_info (
  id  int  primary key auto_increment comment 'id',
  shop_name varchar(64) not null comment '门店名称',
  geom_point geometry not null comment '经纬度’,
  spatial key geom_index(geom_point)
);

```



### 3. 索引的底层原理

```mysql
1.思考
---建表
create table t_emp(id int primary key,name varchar(20),age int);

--插入数据
insert into t_emp values(5,'d',22);
insert into t_emp values(6,'d',22);
insert into t_emp values(7,'e',21);
insert into t_emp values(1,'a',23);
insert into t_emp values(2,'b',26);
insert into t_emp values(3,'c',27);
insert into t_emp values(4,'a',32);
insert into t_emp values(8,'f',53);
insert into t_emp values(9,'v',13);

--查询
select * from t_emp;

我们发现，插入数据的时候，id是不按顺序插入的，但是select表时却发现MySQL自动为我们排好序了，这是为什么？
第一层SQL优化：按照主键为数据进行一次排序
第二层SQL优化：页管理。将几个数据存储到一页中。将这些页建立一个页目录。页目录记录的是每一页的第一个数据的id。一页默认16KB。页目录的存储空间也是一页的大小，16KB。
```

### 4. B+树

```
B+Tree是在B-Tree（B树，不是B减树，-是连接符，没有B减树这个东西）基础上的一种优化，使其更适合实现外存储索引结构，InnoDB存储引擎就是用B+Tree实现其索引结构。

从上一节中的**B-Tree结构图中可以看到每个节点中不仅包含数据的key值，还有数据值**。而每一个页的存储空间是有限的，如果data数据较大时将会导致每个节点（即一个页）能存储的key的数量很小，当存储的数据量很大时同样会导致B-Tree的深度较大，增大查询时的磁盘I/O次数，进而影响查询效率。在B+Tree中，所有数据记录节点都是按照键值大小顺序存放在同一层的**叶子节点**上，而**非叶子节点上只存储key值（主键）信息和指针信息**，这样可以大大加大每个节点存储的key值数量，降低B+Tree的高度。

B+Tree相对于B-Tree有几点不同：

1. 非叶子节点只存储键值信息。
2. 所有叶子节点之间都有一个链指针。
3. 数据记录都存放在叶子节点中。

- InnoDB存储引擎中页的大小为16KB，一般表的主键类型为INT（占用4个字节）或BIGINT（占用8个字节），指针类型也一般为4或8个字节，也就是说一个页（B+Tree中的一个节点）中大概存储16KB/(8B+8B)=1K个键值（因为是估值，为方便计算，这里的K取值为〖10〗^3）。也就是说一个深度为3的B+Tree索引可以维护10^3 * 10^3 * 10^3 = 10亿 条记录。 

- 实际情况中每个节点可能不能填充满，因此在数据库中，B+Tree的高度一般都在2~4层。Mysql的InnoDB存储引擎在设计时是将根节点常驻内存的，也就是说查找某一键值的行记录时最多只需要1~3次磁盘I/O操作。
```

### 5. 聚簇索引和非聚簇索引

```
- 聚簇索引： 将数据存储与索引放到了一块，索引结构的叶子节点保存了行数据
- 非聚簇索引：将数据与索引分开存储，索引结构的叶子节点指向了数据对应的位置

```

`注意`:**在innodb中，在聚簇索引之上创建的索引称之为辅助索引，非聚簇索引都是辅助索引，像复合索引、前缀索引、唯一索引。辅助索引叶子节点存储的不再是行的物理位置，而是主键值，辅助索引访问数据总是需要二次查找**。

1. InnoDB中

- InnoDB使用的是聚簇索引，将主键组织到一棵B+树中，而行数据就储存在叶子节点上，若使用"where id = 14"这样的条件查找主键，则按照B+树的检索算法即可查找到对应的叶节点，之后获得行数据。

- 若对Name列进行条件搜索，则需要两个步骤：第一步在辅助索引B+树中检索Name，到达其叶子节点获取对应的主键。第二步使用主键在主索引B+树中再执行一次B+树检索操作，最终到达叶子节点即可获取整行数据。（**重点在于通过其他键需要建立辅助索引**）
- **聚簇索引默认是主键**，如果表中没有定义主键，InnoDB 会选择一个**唯一且非空的索引**代替。如果没有这样的索引，InnoDB 会**隐式定义一个主键（类似oracle中的RowId）**来作为聚簇索引。如果已经设置了主键为聚簇索引又希望再单独设置聚簇索引，必须先删除主键，然后添加我们想要的聚簇索引，最后恢复设置主键即可。

2. MYISAM

- MyISAM使用的是非聚簇索引，**非聚簇索引的两棵B+树看上去没什么不同**，节点的结构完全一致只是存储的内容不同而已，主键索引B+树的节点存储了主键，辅助键索引B+树存储了辅助键。表数据存储在独立的地方，这两颗B+树的叶子节点都使用一个地址指向真正的表数据，对于表数据来说，这两个键没有任何差别。由于**索引树是独立的，通过辅助键检索无需访问主键的索引树**。

### 6. 关于索引的一些面试题

1. 每次使用辅助索引检索都要经过两次B+树查找，看上去聚簇索引的效率明显要低于非聚簇索引，这不是多此一举吗？**聚簇索引的优势在哪？**

- 1.由于行数据和聚簇索引的叶子节点存储在一起，同一页中会有多条行数据，访问同一数据页不同行记录时，已经把页加载到了Buffer中（缓存器），再次访问时，会在内存中完成访问，不必访问磁盘。这样主键和行数据是一起被载入内存的，找到叶子节点就可以立刻将行数据返回了，如果按照主键Id来组织数据，获得数据更快。

- 2.辅助索引的叶子节点，存储主键值，而不是数据的存放地址。好处是当行数据发生变化时，索引树的节点也需要发生变化；或者是我们需要查找的数据，在上一次IO读写的缓存中没有，需要发生一次新的IO操作时，可以避免对辅助索引的维护工作，只需要维护聚簇索引树就好了。另一个好处是，因为辅助索引存放的是主键值，减少了辅助索引占用的存储空间大小。

2. 聚簇索引需要注意什么?

- 当使用主键为聚簇索引时，主键最好不要使用uuid，因为uuid的值太过离散，不适合排序且可能出现新增加记录的uuid，会插入在索引树中间的位置，导致索引树调整复杂度变大，消耗更多的时间和资源。
- 建议使用int类型的自增，方便排序并且默认会在索引树的末尾增加主键值，对索引树的结构影响最小。而且，主键值占用的存储空间越大，辅助索引中保存的主键值也会跟着变大，占用存储空间，也会影响到IO操作读取到的数据量。

3.  为什么主键通常建议使用自增id

- 聚簇索引的数据的物理存放顺序与索引顺序是一致的，即：只要索引是相邻的，那么对应的数据一定也是相邻地存放在磁盘上的。如果主键不是自增id，那么可以想象，它会干些什么，不断地调整数据的物理地址、分页，当然也有其他一些措施来减少这些操作，但却无法彻底避免。但，如果是自增的，那就简单了，它只需要一页一页地写，索引结构相对紧凑，磁盘碎片少，效率也高。

4.  什么情况下无法利用索引呢?

   - 1.查询语句中使用LIKE关键字
     在查询语句中使用 LIKE 关键字进行查询时，**如果匹配字符串的第一个字符为“%”，索引不会被使用**。如果“%”不是在第一个位置，索引就会被使用。

   - 2.查询语句中使用多列索引
     多列索引是在表的多个字段上创建一个索引，只有查询条件中使用了这些字段中的第一个字段（即最左前缀原则），索引才会被使用。


   - 3.查询语句中使用OR关键字
     查询语句只有OR关键字时，如果OR前后的两个条件的列都是索引，那么查询中将使用索引。如果OR前后有一个条件的列不是索引，那么查询中将不使用索引。

### 7. 存储引擎

```mysql
数据库存储引擎是数据库底层软件组织，数据库管理系统（DBMS）使用数据引擎进行创建、查询、更新和删除数据。
不同的存储引擎提供不同的存储机制、索引技巧、锁定水平等功能。现在许多不同的数据库管理系统都支持多种不同的数据引擎。MySQL的核心就是存储引擎。
用户可以根据不同的需求为数据表选择不同的存储引擎
可以使用 SHOW ENGINES 命令 可以查看Mysql的所有执行引擎我们 可以到 默认的执行引擎是innoDB 支持事务，行级锁定和外键。
MyISAM：Mysql 5.5之前的默认数据库引擎，最为常用。拥有较高的插入，查询速度，但不支持事务
InnoDB：事务型速记的首选引擎，支持ACID事务，支持行级锁定，MySQL5.5成为默认数据库引擎
Memory： 所有数据置于内存的存储引擎，拥有极高的插入，更新和查询效率。但是会占用和数据量成正比的内存空间。并且其内容会在MYSQL重新启动是会丢失。
Archive ：非常适合存储大量的独立的，作为历史记录的数据。因为它们不经常被读取。Archive 拥有高效的插入速度，但其对查询的支持相对较差
Federated ：将不同的 MySQL 服务器联合起来，逻辑上组成一个完整的数据库。非常适合分布式应用
-- 查询当前数据库支持的存储引擎：
show engines;
 
-- 查看当前的默认存储引擎：
show variables like ‘%storage_engine%’;

-- 查看某个表用了什么引擎(在显示结果里参数engine后面的就表示该表当前用的存储引擎): 
show create table student; 
 
-- 创建新表时指定存储引擎：
create table(...) engine=MyISAM;
 
-- 修改数据库引擎
alter table student engine = INNODB;
alter table student engine = MyISAM;

-- 修改MySQL默认存储引擎方法
1. 关闭mysql服务 
2. 找到mysql安装目录下的my.ini文件： 
3. 找到default-storage-engine=INNODB 改为目标引擎，
   如：default-storage-engine=MYISAM 
4. 启动mysql服务

```









