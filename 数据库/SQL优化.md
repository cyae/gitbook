---
date created: 2023-03-06 21:54
date updated: 2023-03-08 19:37
---

#SQL

![](http://n.sinaimg.cn/sinakd20221018s/291/w1080h811/20221018/2b5c-0588511925c5d60f774f430fb9223f73.png)

---

## 问题定位

### 1. 查看慢查询日志

```sql
show variables like ’slow_query_log%’

慢查询阈值
show variables like ’long_query_time’
```

![](http://n.sinaimg.cn/sinakd20221018s/13/w593h220/20221018/8362-46c21cca8389b221009f2a289d442743.png)

![](http://n.sinaimg.cn/sinakd20221018s/752/w558h194/20221018/93a0-711839964c8db677ade3e09d5fc0ee01.png)

### 2. explain分析SQL执行计划

![](http://n.sinaimg.cn/sinakd20221018s/452/w1080h172/20221018/e91b-4089b5bb0d571131be047ba9601f9252.png)

#### 2.1 type

type表示连接类型，查看索引执行情况的一个重要指标。以下性能从好到坏依次：system > const > eq_ref > ref > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL

- **system**：这种类型要求数据库表中只有一条数据，是_const_类型的一个特例，一般情况下是不会出现的
- **const**：通过一次索引就能找到数据，一般用于主键或唯一索引作为条件，这类扫描效率极高，速度非常快
- **eq_ref**：常用于主键或唯一索引扫描，一般指使用主键的关联查询
- **ref** : 常用于非主键和唯一索引扫描
- **ref_or_null**：这种连接类型类似于_ref_，区别在于MySQL会额外搜索包含NULL值的行
- **index_merge**：使用了索引合并优化方法，查询使用了两个以上的索引。
- **unique_subquery**：类似于_eq_ref_，条件用了`in`子查询
- **index_subquery**：区别于_unique_subquery_，用于非唯一索引，可以返回重复值
- **range**：常用于范围查询，比如：`between ... and` 或 `in` 等操作
- **index**：全索引扫描
- **ALL**：全表扫描

#### 2.2 rows

该列表示MySQL估算要找到我们所需的记录，需要读取的行数。对于InnoDB表，此数字是**估计值**，并非一定是个准确值。

#### 2.3 filtered

该列是一个百分比的值，表里符合条件的记录数的百分比。简单点说，这个字段表示存储引擎返回的数据在经过过滤后，剩下满足条件的记录数量的比例。

#### 2.4 extra

该字段包含有关MySQL如何解析查询的其他信息，它一般会出现这几个值：

- **Using filesort**：表示按文件排序，一般是在指定的排序和索引排序不一致的情况才会出现。一般见于`order by`语句
- **Using index**：表示是否用了**覆盖索引**。
- **Using temporary**: 表示是否使用了**临时表**,性能特别差，需要重点优化。一般多见于`group by`语句，或者`union`语句。
- **Using where**: 表示使用了`where`条件过滤.
- **Using index condition**：MySQL5.6之后新增的**索引下推**。在存储引擎层进行数据过滤，而不是在服务层过滤，利用索引现有的数据减少回表的数据。

#### 2.5 key

该列表示实际用到的索引。一般配合`possible_keys`列一起看。

### 3. profile分析执行资源消耗

`explain`只是看到SQL的预估执行计划，如果要了解SQL真正的执行线程状态及消耗的时间，需要使用`profiling`。开启`profiling`参数后，后续执行的SQL语句都会记录其资源开销，包括_IO_，_上下文切换_，_CPU_，_内存_等等，我们可以根据这些开销进一步分析当前慢SQL的瓶颈再进一步进行优化。

![](http://n.sinaimg.cn/sinakd20221018s/68/w605h263/20221018/5392-e971a950b338b49a62a5f08e902fc7d2.png)

![](http://n.sinaimg.cn/sinakd20221018s/713/w881h632/20221018/13ed-b2fb4a2b18fab398e1d8eb092051dd25.png)

`show profiles`会显示最近发给服务器的多条语句，条数由变量`profiling_history_size`定义，默认是15。如果我们需要看单独某条SQL的分析，可以`show profile`查看最近一条SQL的分析，也可以使用`show profile for query id`（其中`id`就是`show profiles`中的`QUERY_ID`）查看具体一条的SQL语句分析。

![](http://n.sinaimg.cn/sinakd20221018s/639/w852h587/20221018/c234-228c5bbbb1fafd4579542427bb398374.png)

### 4. optimizer trace追溯执行流程

`profile`只能查看到SQL的执行耗时，但是无法看到SQL真正执行的过程信息，即不知道MySQL**优化器**是如何**选择执行计划**。这时候，我们可以使用`Optimizer Trace`，它可以跟踪执行语句的解析优化执行的全过程。

![](http://n.sinaimg.cn/sinakd20221018s/529/w1080h249/20221018/4814-33900f18effd8bdba9b749def7100c4f.png)

![](http://n.sinaimg.cn/sinakd20221018s/208/w1080h728/20221018/7edc-f47c47c6aebcd3d230c3418b7ca4efee.png)

## 问题处理

### 1. 隐式转换

我们创建一个用户`user`表:

```sql
CREATE  TABLE  user (  
	id  int(11)  NOT  NULL  AUTO_INCREMENT,   
    userId  varchar(32)  NOT  NULL,   
    age   varchar(16)  NOT  NULL,   
    name  varchar(255)  NOT  NULL,   
    PRIMARY  KEY (id),   
    KEY  idx_userid (
      userId
    )  USING  BTREE
)  ENGINE = InnoDB  DEFAULT  CHARSET = utf8;
```

userId字段为字串类型，是B+树的普通索引，如果查询条件传了一个数字过去，会导致索引失效。如下：
![](http://n.sinaimg.cn/sinakd20221018s/498/w1080h218/20221018/b9b0-fb6d6f54c4e8be1b004070e7b4b96b91.png)

如果给数字加上'',也就是说，传的是一个字符串呢? 当然是走索引，如下图：
![](http://n.sinaimg.cn/sinakd20221018s/480/w1080h200/20221018/6e16-45218e5c907346df3d87929353aedd6b.png)

> 为什么第一条语句未加单引号就不走索引了呢？
>
> 这是因为不加单引号时，是字符串跟数字的比较，它们类型不匹配，MySQL会做隐式的类型转换，把它们转换为浮点数再做比较。隐式的类型转换，索引会失效。

### 2. 最左匹配

MySQl建立**联合索引**时，会遵循**最左前缀匹配**的原则，即最左优先。如果你建立一个`(a,b,c)`的联合索引，相当于建立了`(a)`、`(a,b)`、`(a,b,c)`三个索引。

假设有以下表结构：

```sql
CREATE  TABLE  user (  
	id  int(11)  NOT  NULL  AUTO_INCREMENT,   
    user_id  varchar(32)  NOT  NULL,   
    age   varchar(16)  NOT  NULL,   
    name  varchar(255)  NOT  NULL,   
    PRIMARY  KEY (id),   
    KEY  idx_userid_name (
      user_id, name
    )  USING  BTREE
)  ENGINE = InnoDB  DEFAULT  CHARSET = butf8;
```

假设有一个联合索引`idx_userid_name`，我们现在执行以下SQL，如果查询列是`name`，索引是无效的：

```sql
explain select * from user where name ='捡田螺的小男孩';
```

![](http://n.sinaimg.cn/sinakd20221018s/462/w1080h182/20221018/77aa-8dbfcac025f0ed24a772803a69b767df.png)

因为查询条件列`name`不是联合索引`idx_userid_name`中的第一个列，不满足最左匹配原则，所以索引不生效。在联合索引中，只有查询条件满足最左匹配原则时，索引才正常生效。如下，查询条件列是`user_id`
![](http://n.sinaimg.cn/sinakd20221018s/501/w1080h221/20221018/96c4-a4696c24f1ebd85cb5e147995e9ac106.png)

### 3. 深分页问题

`limit`深分页问题，会导致慢查询，应该大家都司空见惯了吧。

假设有表结构如下：

```sql
CREATE  TABLE  account (  
	id  int(11)  NOT  NULL  AUTO_INCREMENT  COMMENT  '主键Id',
    name  varchar(255)  DEFAULT  NULL  COMMENT  '账户名',
    balance  int(11)  DEFAULT  NULL  COMMENT  '余额',
	create_time  datetime  NOT  NULL  COMMENT  '创建时间',   
    update_time  datetime  NOT  NULL  ON  UPDATE  CURRENT_TIMESTAMP  COMMENT  '更新时间',
    PRIMARY  KEY (id),   
    KEY  idx_name (name),   
    KEY  idx_create_time (
      create_time
    )  //索引
) ENGINE=InnoDB AUTO_INCREMENT=DEFAULT CHARSET=utf8 ROW_FORMAT=REDUNDANT COMMENT='账户表';
```

#### 深分页为什么慢呢？

```sql
select  id, name, balance  
from  account  
where  create_time > '2020-09-19' 
limit  100000, 10;
```

这个SQL的执行流程酱紫：

1. 通过普通二级索引树`idx_create_time`，过滤`create_time`条件，找到满足条件的主键`id`。
2. 通过主键`id`，回到`id`主键索引树，找到满足记录的行，然后取出需要展示的列（回表过程）
3. 扫描满足条件的`100010`行，然后扔掉前`100000`行，返回。

![](http://n.sinaimg.cn/sinakd20221018s/468/w1080h188/20221018/714b-a2859b512b8914340dbed9e47797e136.png)

因此，`limit`深分页，导致SQL变慢原因有两个：

- `limit`语句会先扫描offset+n行，然后再丢弃掉前offset行，返回后n行数据。也就是说`limit 100000,10`，就会扫描100010行，而`limit 0,10`，只扫描10行。
- `limit 100000,10` 扫描更多的行数，也意味着**回表**更多的次数。

#### 💡如何优化深分页问题?

我们可以通过**减少回表次数**来优化。一般有标签记录法和延迟关联法。

##### 标签记录法

> 就是标记一下上次查询到哪一条了，下次再来查的时候，从该条开始往下扫描。就好像看书一样，上次看到哪里了，你就折叠一下或者夹个书签，下次来看的时候，直接就翻到啦。

假设上一次记录到100000，则SQL可以修改为：

```sql
select  id,name,balance 
FROM account
where id > 100000 
limit 10;
```

这样的话，后面无论翻多少页，性能都会不错的，因为命中了id索引。但是这种方式有**局限性**：需要一种类似**连续自增**的字段。

##### 延迟关联法

延迟关联法，就是把条件转移到主键索引树，然后减少回表。如下

```sql
select   acct1.id, acct1.name, acct1.balance  
FROM  account  acct1
INNER JOIN (
	SELECT  a.id  
	FROM  account  a  
	WHERE  a.create_time  >  '2020-09-19' 
	limit  100000,  10
)  AS  acct2  
on  acct1.id = acct2.id;
```

优化思路就是，先通过`idx_create_time`二级索引树查询到满足条件的主键ID，再与原表通过主键ID**内连接**，这样后面直接走主键索引了，同时也减少了回表。

### 4. in元素过多

如果使用了`in`，即使后面的条件加了索引，也要注意`in`后面的元素不要过多。`in`元素一般建议不要超过`200`个，如果超过了，建议分组，每200一组进行。

反例:

```sql
select user_id,name 
from user 
where user_id 
in (1,2,3,...,1000000); 
```

如果我们对`in`的条件不做任何限制的话，该查询语句一次性可能会查询出非常多的数据，很容易导致接口超时。

尤其有时候，我们是用的子查询，**in后面的子查询结果集大小未知**，更容易采坑. 如下这种子查询：

```sql
select * 
from user 
where user_id 
in (
	select author_id 
	from artilce 
	where type = 1
);
```

如果`type = 1`有一千，甚至上万个呢？肯定是慢SQL。索引一般建议分批进行，一次200个，比如：

```sql
select user_id,name 
from user 
where user_id 
in (1,2,3...200);
```

#### in查询为什么慢呢？

> 这是因为`in`查询在MySQL底层是通过n*m笛卡尔积的方式去搜索，类似`union`。
>
> `in`查询在进行cost代价计算时（代价 = 元组数 * IO平均值），是通过将`in`包含的数值，一条条去查询获取元组数的，因此这个计算过程会比较的慢，所以MySQL设置了个临界值(`eq_range_index_dive_limit`)，5.6之后超过这个临界值后该列的cost就不参与计算了。因此会导致**执行计划选择不准确**。
>
> 默认是`200`，即`in`条件超过了200个数据，会导致`in`的代价计算存在问题，可能会导致Mysql选择的索引不准确。

### 5. order by走filesort导致的慢查询

我们来看下下面这个SQL：

```sql
select  name, age, city  
from  staff  
where  city  =  '深圳' 
order  by  age  
limit  10;
```

它表示的意思就是：查询前10个，来自深圳员工的姓名、年龄、城市，并且按照年龄小到大排序。
![](http://n.sinaimg.cn/sinakd20221018s/478/w1080h198/20221018/95e4-56cfa5571ab4e59a621be4b3be113249.png)

查看`explain`执行计划的时候，可以看到`Extra`这一列，有一个`Using filesort`，它表示用到文件排序。

#### order by文件排序效率为什么低?

大家可以看下这个下面这个图:
![](http://n.sinaimg.cn/sinakd20221018s/507/w1080h227/20221018/6f9d-994469e6cfe6e78e8a7d79c61b314867.png)

`order by`排序，分为_全字段排序_和_rowid排序_。它是拿`max_length_for_sort_data`和结果行数据长度对比，如果结果行数据长度超过`max_length_for_sort_data`这个值，就会走rowid排序，相反，则走全字段排序。

##### rowid排序

rowid排序，一般需要**回表**去找满足条件的数据，所以效率会慢一点。以下这个SQL，使用rowid排序，执行过程是这样：

```sql
select  name, age, city  
from  staff  
where  city  =  '深圳' 
order  by  age  
limit  10;
```

1. MySQL为对应的线程初始化sort_buffer，放入需要排序的`age`字段，以及主键id；
2. 从索引树`idx_city`， 找到第一个满足 `city='深圳'`条件的主键id,假设id为X；
3. 到主键id索引树拿到`id=X`的这一行数据， 取`age`和主键id的值，存到sort_buffer；
4. 从索引树`idx_city`拿到下一个记录的主键id，假设`id=Y`；
5. 重复步骤 3、4 直到`city`的值不等于深圳为止；

前面5步已经查找到了所有`city`为深圳的数据，在sort_buffer中，将所有数据根据`age`进行排序；遍历排序结果，取前10行，并按照id的值回到原表中，取出`city`、`name` 和 `age`三个字段返回给客户端。
![](http://n.sinaimg.cn/sinakd20221018s/797/w1080h517/20221018/ac59-7183c504494f0abf95cc9417fa5bdfcb.png)

##### 全字段排序

同样的SQL，如果是走全字段排序是这样的：

```sql
select  name, age, city  
from  staff  
where  city  =  '深圳' 
order  by  age  
limit  10;
```

1. MySQL 为对应的线程初始化sort_buffer，放入需要查询的`name`、`age`、`city`字段；
2. 从索引树`idx_city`， 找到第一个满足 `city='深圳'`条件的主键 id，假设找到`id=X`；
3. 到主键id索引树拿到`id=X`的这一行数据， 取`name`、`age`、`city`三个字段的值，存到sort_buffer；
4. 从索引树`idx_city` 拿到下一个记录的主键id，假设`id=Y`；
5. 重复步骤 3、4 直到`city`的值不等于深圳为止；

前面5步已经查找到了所有`city`为深圳的数据，在sort_buffer中，将所有数据根据age进行排序；按照排序结果取前10行返回给客户端。
![](http://n.sinaimg.cn/sinakd20221018s/109/w1080h629/20221018/442c-2423eb7b4acb1fc6d4314c7be12104ab.png)

sort_buffer的大小是由一个参数控制的：`sort_buffer_size`。

- 如果要排序的数据小于`sort_buffer_size`，排序在sort_buffer**内存**中完成
- 如果要排序的数据大于`sort_buffer_size`，则借助**磁盘文件**来进行排序。

> 借助磁盘文件排序的话，效率就**更慢**。因为先把数据放入sort_buffer，当快要满时。会排一下序，然后把sort_buffer中的数据，放到临时磁盘文件，等到所有满足条件数据都查完排完，再用**归并算法**把磁盘的临时排好序的小文件，合并成一个有序的大文件。

#### 💡如何优化order by的文件排序

- 因为数据是无序的，所以就需要排序。如果**数据本身是有序**的，那就不会再用到文件排序啦。而**索引数据本身是有序**的，我们通过建立索引来优化`order by`语句。
- 我们还可以通过调整`max_length_for_sort_data`、`sort_buffer_size`等参数优化；

### 6. 索引字段上使用is null， is not null，索引可能失效

表结构:

```sql
CREATE  TABLE  `user`  (  
	id  int(11)  NOT  NULL  AUTO_INCREMENT,   
	card varchar(255)  DEFAULT  NULL,   
	name varchar(255)  DEFAULT  NULL,   
	PRIMARY  KEY (id),   
	KEY  idx_name  (
		name
    )  USING  BTREE,   
    KEY  idx_card  (
	    card
	)  USING  BTREE
)  ENGINE = InnoDB  AUTO_INCREMENT = 2  DEFAULT  CHARSET = utf8;
```

单个`name`字段加上索引，并查询`name`为非空的语句，其实会走索引的，如下:
![](http://n.sinaimg.cn/sinakd20221018s/459/w1080h179/20221018/bdfa-c027c61ce8ea94e9b520124bfa7f7dd2.png)

单个`card`字段加上索引，并查询`name`为非空的语句，其实会走索引的，如下:
![](http://n.sinaimg.cn/sinakd20221018s/501/w1080h221/20221018/343b-f34404014b3578e9b3185d168ecf0fbb.png)

但是它两用`or`连接起来，索引就失效了，如下：
![](http://n.sinaimg.cn/sinakd20221018s/531/w1080h251/20221018/dd32-4bbfc465a1b17dab3bb32fd9cf33e44d.png)

很多时候，也是因为**数据量问题**，导致了MySQL**优化器放弃走索引**。同时，平时我们用`explain`分析SQL的时候，如果`type=range`,要注意一下，因为这个可能因为数据量问题，导致索引无效。

### 7. 索引字段上使用（!= 或者 not in），索引可能失效

假设有表结构：

```sql
CREATE  TABLE  user  (  
	id  int(11)  NOT  NULL  AUTO_INCREMENT,   
	userId  int(11)  NOT  NULL,   
	age int(11)  DEFAULT  NULL,   
	name varchar(255)  NOT  NULL,   
	PRIMARY  KEY (id),   
	KEY  idx_age  (
      age
    )  USING  BTREE
)  ENGINE = InnoDB  AUTO_INCREMENT = 2  DEFAULT  CHARSET = utf8;
```

虽然`age`加了索引，但是使用了`!=`或者，`not in`这些时，索引如同虚设。如下：
![](http://n.sinaimg.cn/sinakd20221018s/445/w1080h165/20221018/31eb-c8ec9431da0e10b02f06187d23f0db52.png)

其实这个也是跟mySQL**优化器**有关，如果优化器觉得即使走了索引，还是需要扫描很多很多行的哈，它觉得不划算，不如直接不走索引。平时我们用 `!=` 或者，`not in`的时候，留点心眼。

### 8. 左右连接，关联的字段编码格式不一样

新建两个表，一个`user`，一个`user_job`:

```sql
CREATE  TABLE  user  (  
	id int(11)  NOT  NULL  AUTO_INCREMENT,   
	name varchar(255)  CHARACTER  SET  utf8mb4  DEFAULT  NULL,   
    age int(11)  NOT  NULL,   
    PRIMARY  KEY (id),   
    KEY  idx_name  (
      name
    )  USING  BTREE
)  ENGINE = InnoDB  AUTO_INCREMENT = 2  DEFAULT  CHARSET = utf8;　　
  
CREATE  TABLE  user_job  (  
	id int(11)  NOT  NULL,   
	userId int(11)  NOT  NULL,   
    job varchar(255)  DEFAULT  NULL,   
    name varchar(255)  DEFAULT  NULL,   
    PRIMARY  KEY (id),   
    KEY  idx_name  (
      name
    )  USING  BTREE
)  ENGINE = InnoDB  DEFAULT  CHARSET = utf8;
```

`user`表的`name`字段编码是`utf8mb4`，而`user_job`表的`name`字段编码为`utf8`。
![](http://n.sinaimg.cn/sinakd20221018s/590/w1038h352/20221018/8f25-629a5776c5eccaf42860bdbc845ed49e.png)

执行左外连接查询, `user_job`表还是走全表扫描，如下：
![](http://n.sinaimg.cn/sinakd20221018s/565/w1080h285/20221018/51ad-1f7efae98f9f207bca51ced374889195.png)

如果把它们的`name`字段改为编码一致，相同的SQL，还是会走索引。
![](http://n.sinaimg.cn/sinakd20221018s/579/w1080h299/20221018/e6a9-9e1abba85a8ec639783104480b2e0792.png)

### 9. group by使用临时表

`group by`一般用于**分组统计**，它表达的逻辑就是根据一定的规则，进行分组。日常开发中，我们使用得比较频繁。如果不注意，很容易产生慢SQL。

#### group by执行流程

假设有表结构：

```sql
CREATE  TABLE  staff  (  
	id bigint(11)  NOT  NULL  AUTO_INCREMENT  COMMENT  '主键id',   
    id_card varchar(20)  NOT  NULL  COMMENT  '身份证号码',   
    name varchar(64)  NOT  NULL  COMMENT  '姓名',   
    age int(4)  NOT  NULL  COMMENT  '年龄',   
    city varchar(64)  NOT  NULL  COMMENT  '城市',   
    PRIMARY  KEY (
	    id
	)
)  ENGINE = InnoDB  AUTO_INCREMENT = 15  DEFAULT  CHARSET = utf8  COMMENT = '员工表';
```

我们查看一下这个SQL的执行计划：

```sql
explain select city ,count(*) as num from staff group by city;
```

![](http://n.sinaimg.cn/sinakd20221018s/491/w1080h211/20221018/20c6-1f2f2461c0ae5b04052c05ee2de43cd9.png)

- `Extra` 这个字段的`Using temporary`表示在执行分组的时候使用了**临时表**
- `Extra` 这个字段的`Using filesort`表示使用了**文件排序**

`group by`是怎么使用到临时表和排序了呢？我们来看下这个SQL的执行流程

```sql
select city ,count(*) as num from staff group by city;
```

创建内存临时表，表里有两个字段`city`和`num`；

全表扫描`staff`的记录，依次取出`city = X`的记录。

- 判断临时表中是否有为`city = X`的行，没有就插入一个记录 (X,1);
- 如果临时表中有`city = X`的行，就将X这一行的num值加 1；

遍历完成后，再根据字段`city`做排序，得到结果集返回给客户端。这个流程的执行图如下：
![](http://n.sinaimg.cn/sinakd20221018s/710/w1048h462/20221018/0e0a-7a0da3a29385ede317cfda8d0e117db0.png)

#### 临时表的排序是怎样的呢？

就是把需要排序的字段，放到sort buffer，排完就返回。在这里注意一点哈，排序分全字段排序和rowid排序:

- 如果是全字段排序，需要查询返回的字段，都放入sort buffer，根据排序字段排完，直接返回
- 如果是rowid排序，只是需要排序的字段放入sort buffer，然后多一次回表操作，再返回。

#### group by可能会慢在哪里？

`group by`使用不当，很容易就会产生慢SQL问题。因为它既用到**临时表**，又默认用到**排序**。有时候还可能用到**磁盘临时表**:

- 如果执行过程中，会发现**内存临时表**大小到达了上限（控制这个上限的参数就是`tmp_table_size`），会把内存临时表转成磁盘临时表。
- 如果数据量很大，很可能这个查询需要的磁盘临时表，就会占用大量的磁盘空间。

#### 💡如何优化group by呢

从哪些方向去优化呢？

- 既然它默认会排序，我们不给它排是不是就行啦。
- 既然临时表是影响`group by`性能的X因素，我们是不是可以不用临时表？

我们一起来想下，执行`group by`语句为什么需要临时表呢？`group by`的语义逻辑，就是统计不同的值出现的个数。如果这个这些值一开始就是有序的，我们是不是直接往下扫描统计就好了，就不用临时表来记录并统计结果啦?

可以有这些优化方案：

- `group by` 后面的字段加索引
- `order by null` 不用排序
- 尽量只使用内存临时表
- 使用`SQL_BIG_RESULT`

### 10.  delete + in子查询不走索引

之前见到过一个生产慢SQL问题，当`delete`遇到`in`子查询时，即使有索引，也是不走索引的。而对应的`select + in`子查询，却可以走索引。

MySQL版本是5.7，假设当前有两张表`account`和`old_account`,表结构如下：

```sql
CREATE  TABLE  old_account  (  
	id int(11)  NOT  NULL  AUTO_INCREMENT  COMMENT  '主键Id', 
    name varchar(255)  DEFAULT  NULL  COMMENT  '账户名', 
    balance int(11)  DEFAULT  NULL  COMMENT  '余额', 
    create_time datetime  NOT  NULL  COMMENT  '创建时间',
    update_time datetime  NOT  NULL  ON  UPDATE  CURRENT_TIMESTAMP  COMMENT  '更新时间',   
    PRIMARY  KEY (id),   
    KEY  idx_name  (
	    name
	)  USING  BTREE
)  ENGINE = InnoDB  AUTO_INCREMENT =  DEFAULT  CHARSET = utf8  ROW_FORMAT = REDUNDANT  COMMENT = '老的账户表';
　　
CREATE  TABLE  account  (  
	id int(11)  NOT  NULL  AUTO_INCREMENT  COMMENT  '主键Id',   
    name varchar(255)  DEFAULT  NULL  COMMENT  '账户名',   
    balance int(11)  DEFAULT  NULL  COMMENT  '余额',   
    create_time datetime  NOT  NULL  COMMENT  '创建时间',   
    update_time datetime  NOT  NULL  ON  UPDATE  CURRENT_TIMESTAMP  COMMENT  '更新时间',   
    PRIMARY  KEY (
	    id
	),   
	KEY  idx_name  (
	    name
	)  USING  BTREE
)  ENGINE = InnoDB  AUTO_INCREMENT = DEFAULT  CHARSET = utf8  ROW_FORMAT = REDUNDANT  COMMENT = '账户表';
```

执行的SQL如下：

```sql
delete from account 
where name 
in (
	select name 
	from old_account
);
```

查看执行计划，发现不走索引：
![](http://n.sinaimg.cn/sinakd20221018s/480/w1080h200/20221018/726e-40f9686d6e412344779ec52cce2c5110.png)

但是如果把`delete`换成`select`，就会走索引。如下：
![](http://n.sinaimg.cn/sinakd20221018s/476/w1080h196/20221018/d7e7-d9b71efd85237b1cdbec7dd1067b9ba6.png)

为什么`select + in`子查询会走索引，`delete + in`子查询却不会走索引呢？
我们执行以下SQL看看：

```sql
explain  select  *  from  account  where  name  in  (select  name  from  old_account);

show  WARNINGS;  //可以查看优化后,最终执行的sql
```

结果如下：

```sql
select `test2`.`account`.`id`
AS `id`, `test2`.`account`.`name`
AS `name`, `test2`.`account`.`balance`
AS `balance`, `test2`.`account`.`create_time`
AS `create_time`, `test2`.`account`.`update_time`
AS `update_time`
from `test2`.`account`
semi join (
	`test2`.`old_account`
)
where (
	`test2`.`account`.`name` = `test2`.`old_account`.`name`
)
```

可以发现，实际执行的时候，MySQL对`select in`子查询做了优化，把子查询改成`join`的方式，所以可以走索引。但是很遗憾，对于`delete in`子查询，MySQL却没有对它做这个优化。

日常开发中，大家注意一下这个场景

### 11. 为什么一般推荐MySQL记录条数不超过5000万?

- 表结构更改, 索引更改耗时, STW
- 主从同步耗时
- 索引性能下降

### 12. 千万级数据多条件联合查询优化

某表内包含千万条记录, 现需要按照多条件进行筛选, SQL如下

```SQL
SELECT * FROM t_table WHERE 1 = 1
AND Condition1
AND Condition2
AND Condition3
AND Condition4
AND Condition5
AND Condition6
...
AND ConditionN
ORDER BY ...
DESC ...
LIMIT ..., ...;
```

其中每个条件`ConditionX`又是表中字段的操作集合.如果按照普通查询方式, 必然造成慢SQL.

#### 对策

上述SQL等价于

```SQL
SELECT * FROM t_table WHERE 1 = 1
AND !(!Condition1)
AND !(!Condition2)
AND !(!Condition3)
AND !(!Condition4)
AND !(!Condition5)
AND !(!Condition6)
...
AND !(!ConditionN)
ORDER BY ...
DESC ...
LIMIT ..., ...;
```

即要求所有反条件`!ConditionX`都为假, 也即某条记录只要任一`!ConditionX`为真, 就不包含在结果集里.

因此考虑分表, 维护一张新表`t_table_condition`, 包含原表`t_table`的主键和其反条件值`!ConditionX`, 查询时只需从新表里查`!ConditionX<>0`即可.

这样好处在于将单次复杂query的耗时均摊到每次insert里.

#### 拓展

新表`t_table_condition`的反条件值`!ConditionX`应该表示多种状态的可能组合, 为了节省空间, 使用二进制压缩的思想, 如`!ConditionX = 1010`表示此记录的第二种和第四种条件组合为假
