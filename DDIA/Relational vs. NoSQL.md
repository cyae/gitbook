>关系型数据库最初的使用背景是 商业数据处理, 包括交易处理和批处理
>随着硬件和网络连接能力的提升, 需要新的可扩容, 支持数据结构的存储

## 1. 一对多框架

面向对象语言OOP 和 关系型存储 之间有着天然的隔阂: 需要引入**中间层**充当 *类对象* 和 *行记录* 间的翻译. 常见的ORM包括iBatis, Hibnernate, jooq等

但是关系型存储模型无法尽善尽美, 比如**一对多的数据存储**, 对象的属性是**无限**的集合(如某人的工作经历可以有多份, **工作经历**列表的大小是不确定的). 

![[Pasted image 20230212214619.png]]

那么做如下考察:
- 使用关系型存储模型. 为防止单页过大, 需要针对该属性新建表, 并建立与原表属性的外键
	- 关系型存储可以保证事务, 但有时不需要严格满足ACID
	- 外键导致的一系列问题
- 有些关系型存储模型将列表属性表示为JSON/XML格式的文件, 并支持文件内部的索引
- NoSQL: 把列表属性当作JSON/XML文件, ORM映射由开发者确定, 存储层不提供索引

若干文档型数据库提供了一对多(树形)数据的存储模型, 如MongoDB
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

此时, 关系型数据库表现最好:
- 可以在存储层使用`join`做关联查询
- 一对多存储模型不支持join, 需要在应用层手动模拟 

## 3. 多对多框架(混合框架)

现实应用中, 一对多和多对一往往混合在一起:
- 虚线方框内是一对多的个人信息, 可以用NOSQL
- 虚线框间存在 有限交集 或 互相引用 的多对一场景, 往往需要支持`join`, 可以用关系型DB

![[Pasted image 20230212214957.png]]
