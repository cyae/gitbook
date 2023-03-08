---
date created: 2023-03-08 14:10
---

#索引 #数据存储

> 关系型数据库最初的使用背景是 商业数据处理, 包括交易处理和批处理
> 随着硬件和网络连接能力的提升, 需要新的可扩容, 支持数据结构的存储

## 1. 一对多框架

面向对象语言OOP 和 关系型存储 之间有着天然的隔阂: 需要引入**中间层**充当 _类对象_ 和 _行记录_ 间的翻译. 常见的ORM包括iBatis, Hibnernate, jooq等.

但是关系型存储模型无法尽善尽美, 比如**一对多的数据存储**, 对象的属性是**无限**的集合(如某人的工作经历可以有多份, **工作经历**列表的大小是不确定的).

![[Pasted image 20230212214619.png]]

那么做如下考察:

- 使用传统的关系型存储模型. 为防止单页过大, 需要针对该属性新建表, 并建立与原表属性的外键
  - 关系型存储可以保证事务, 但有时不需要严格满足ACID
  - 外键导致的一系列问题
- 现代关系型存储模型将列表属性表示为JSON/XML格式的文件, 并支持文件内部的索引
- NoSQL: 把列表属性当作JSON/XML文件, ORM映射由开发者确定

文档型数据库提供了一对多(树形)数据的底层存储模型, 如以下是MongoDB存储的简历, 会表示为二进制的BSON:

```json
{ 
	"user_id": 251, 
	"first_name": "Bill", 
	"last_name": "Gates", 
	"summary": "Co-chair of the Bill & Melinda Gates... Active blogger.", 
	"region_id": "us:91", 
	"industry_id": 131, 
	"photo_url": "/p/7/000/253/05b/308dd6e.jpg",
	"positions": [ 
		{"job_title": "Co-chair", "organization": "Bill & Melinda Gates Foundation"}, 
		{"job_title": "Co-founder, Chairman", "organization": "Microsoft"} 
	], 
	"education": [ 
		{"school_name": "Harvard University", "start": 1973, "end": 1975}, 
		{"school_name": "Lakeside School, Seattle", "start": null, "end": null} 
	], 
	"contact_info": { 
		"blog": "http://thegatesnotes.com", 
		"twitter": "http://twitter.com/BillGates" 
	}
}
```

## 2. 多对一框架

有时, 对象的属性只能来自**有限**的集合, 如居住地只能是所有国家中的一个, 工作领域只能是所有工种的一个, 这些有限集合通常表示成**可选列表**. 这形成了**多个对象对应一个集合元素**的多对一映射(张三李四都住在中国, Tom和Marry和Jack都从事计算机工作)

此时, 关系型数据库表现更好:

- 可以在存储层使用`join`做关联查询
- 一对多存储模型不支持`join`, 需要在应用层手动模拟

## 3. 多对多框架(混合框架)

现实应用中, 一对多 和 多对一 往往混合在一起:

- **虚线框内**是一对多的个人信息, 可以用文档模型
- **虚线框间**存在 有限交集 的多对一场景, 往往需要支持`join`操作, 可以用关系型DB
- **虚线框间**还可能存在 互相引用 的多对多场景, 这时需要根据具体需求确定底层存储模型, 或者使用图DB

![[Pasted image 20230212214957.png]]

## 4. 组织策略(schema)灵活性(flexibility)

对于关系模型, 在表创建的一刻, 数据的schema就按照字段规定好了. 如果有更改schema的需求, 要使用`ALTER TABLE`, 这会挤占业务资源.

对于文档型模型和支持XML/JSON的关系模型, 数据组织方式可以延迟到业务层再确定, 这尤其适用于:

- 存储种类过多, 为每个类建立一张表开销太大
- 数据的组织方式由外部服务确定

## 5. 数据局部性(locality)

对于文档模型, 由于数据被表示为通用的JSON/XML, 因此可以将常用数据放在一个文档里, 减少over-fetching. 这要求:

- 文档大小不能太大
- 尽量避免增加文档大小的写操作

对于关系模型, 需要关联多个表 & 多次IO & 磁盘检索, 数据局部性较差. 但也可以单独维护局部性, 比如Spanner, Oracle, Bigtable.

---

## 趋势

总的而言, 基本存储单元的形式在统一:

- 关系型模型逐步支持JSON/XML的结构与内部索引
- 文档型模型逐步支持并优化`join`等多表操作
