
以前的项目中很少去思考SQL解析这个事情，即使在saas系统或者分库分表的时候有涉及到也会有专门的处理方案，这些方案也对使用者隐藏了实现细节。

而最近的这个数据项目里面却频繁涉及到了对SQL的处理，原来只是简单地了解Druid的SqlParser模块就可以解决，慢慢地问题变得越来越复杂，直到某天改动自己写的SQL处理的代码很痛苦的时候，意识到似乎有必要更加地了解一下相关的内容才行。

在了解学习的过程中，发现学习使用SqlParser还是得先了解ast（抽象语法树）这个概念，一搜索相关内容要么是编译原理相关的知识，要么是JavaScript的示例，光看Druid提供的SqlParser相关的Wiki文档又似懂非懂，不知道从哪里下手。

不管怎么样，看了不少碎片化的相关内容以后也收获了一些东西，这里记录下来。

## 为什么要先了解ast？

ast全称是abstract syntax tree，中文直译抽象语法树。

原先我觉得要使用SqlParser就照着wiki上的代码步骤拷过来就好了呗，也确实如此，它快速解决了我的问题。可是正如上面所说，希望你的相关代码写得更好一点，或者更理解它是在干吗了解了ast会有不少的帮助。

SQL解析，本质上就是把SQL字符串给解析成ast，也就是说SqlParser的入参是SQL字符串，结果就是一个ast。你怎么使用这个ast结果又是另外一回事，你可以修改ast，也可以添加点东西等等，但整个过程都是围绕着ast这个东西。

![](https://img2018.cnblogs.com/blog/1209816/201904/1209816-20190410234736398-1820516882.jpg)

## 什么是ast？

上面提了好几次ast，那ast又是个什么东西呢？

参照维基百科的说法，在计算机科学领域内，ast表示的是你写的编程语言源代码的抽象语法结构。如图：

![](https://img2018.cnblogs.com/blog/1209816/201904/1209816-20190410235708924-2017257369.jpg)

左边是一个非常简单的编程语言源代码：1 + 2，做了一个加法计算，而当它被解析成ast以后如右边的图所示。我们可以看到ast存在三个节点，顶部的 + 表示一个加法节点，这个表达式组合了1、2两个数值节点，由这三个组合在一起的节点就组成了1+2这样的语法结构。

我们看到ast很清晰地用数据结构表示出了字符串源代码，ast的每一个节点均表示源代码当中的一个语法结构。反过来思考一下，我们可以知道源代码解析出来的ast是由很多这样简单的语法结构组合而成的，也就形成了一个复杂的语法树。下面我们看一个稍微复杂一点的，来自维基百科的示例

源代码：


1 while b ≠ 0
2   if a > b
3     a = a − b
4   else
5     b = b − a
6 return a


语法树：

![](https://img2018.cnblogs.com/blog/1209816/201904/1209816-20190411001137101-75806028.png)

这个语法树也清晰地表示的源代码程序，主要由一个while语法和if/else语法以及一些变量之类的组成。

到这里，似乎对源代码和ast有了一个简单的概念，但是还是存在困惑，我为什么要把好好的代码搞成这样？它有什么用？如果只是修改语法，我用正则表达式修改字符串不是简单吗？

确实，有的时候直接处理字符串会是更快速更好的解决方式，但是当源程序语法非常复杂的时候字符串处理的复杂度已经不是一个简单的事了。而ast则把这些字符串变成结构化的数据了，你可以精确地知道一段代码里面有哪些变量名，函数名，参数等，你可以非常精准地处理，相对于字符串处理来说，遍历数据大大降低的处理难度。而ast也常常用在如IDE中错误提示、自动补全、编译器、语法翻译、重构、代码混淆压缩转换等。

了解ast可以参考文章：

[https://mp.weixin.qq.com/s/UYzwVRPFas6hwe2U7R0eIg](https://mp.weixin.qq.com/s/UYzwVRPFas6hwe2U7R0eIg)

[https://www.cnblogs.com/jacksplwxy/p/10676578.html](https://www.cnblogs.com/jacksplwxy/p/10676578.html)

[https://blog.csdn.net/fei33423/article/details/79452922](https://blog.csdn.net/fei33423/article/details/79452922)

[https://www.jianshu.com/p/6a2f4ae4e099](https://www.jianshu.com/p/6a2f4ae4e099)

[https://en.wikipedia.org/wiki/Abstract_syntax_tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree)

[https://en.wikipedia.org/wiki/Parse_tree#Constituency-based_parse_trees](https://en.wikipedia.org/wiki/Parse_tree#Constituency-based_parse_trees)

## SqlParser

我们知道了ast是一种结构化的源代码表示，那针对SQL来说ast就是把SQL语句用结构化的数据来表示了。而SqlParser也就是把SQL解析成ast，这个解析过程则被SqlParser做了隐藏，我们不需要去实现这样一个字符串解析过程。

由此可见，我们需要了解两方面内容：

1、怎么用SqlParser把SQL语句解析成ast；

2、SqlParser解析出来的ast是什么样的一个结构。

下面需要一点代码来说明，所以先引入一下maven依赖


1 <dependency>
2      <groupId>com.alibaba</groupId>
3      <artifactId>druid</artifactId>
4      <version>1.1.12</version>
5 </dependency>


### 解析成ast

解析语句相对简单，wiki上直接有示例，如：

String dbType = JdbcConstants.MYSQL;
List<SQLStatement> statementList = SQLUtils.parseStatements(sql, dbType);

SQLUtils的parseStatements方法会把你传入的SQL语句给解析成SQLStatement对象集合，每一个SQLStatement代表一条完整的SQL语句，如：

SELECT id FROM user WHERE status = 1

多个SQLStatement如：

SELECT id FROM user WHERE status = 1;
SELECT id FROM order WHERE create_time > '2018-01-01'

一般上我们只处理一条语句。

### ast的结构

SQLStatement表示一条SQL语句，我们知道常见的SQL语句有CRUD四种操作，所以SQLStatement会有四种主要实现类，如：

![复制代码](https://common.cnblogs.com/images/copycode.gif)

class SQLSelectStatement implements SQLStatement {
    SQLSelect select;
}
class SQLUpdateStatement implements SQLStatement {
    SQLExprTableSource tableSource;
     List<SQLUpdateSetItem> items;
     SQLExpr where;
}
class SQLDeleteStatement implements SQLStatement {
    SQLTableSource tableSource; 
    SQLExpr where;
}
class SQLInsertStatement implements SQLStatement {
    SQLExprTableSource tableSource;
    List<SQLExpr> columns;
    SQLSelect query;
}

![复制代码](https://common.cnblogs.com/images/copycode.gif)

这里我们以SQLSelectStatement来说明，ast既然是SQL的语法结构表示，我们先看一下ast和SQL select语法的主要对应结构

SQLSelectStatement包含一个SQLSelect，SQLSelect包含一个SQLSelectQuery，都是组成的关系。SQLSelectQuery有主要的两个派生类，分别是SQLSelectQueryBlock和SQLUnionQuery。

![复制代码](https://common.cnblogs.com/images/copycode.gif)

class SQLSelect extends SQLObjectImpl { 
    SQLWithSubqueryClause withSubQuery;
    SQLSelectQuery query;
}

interface SQLSelectQuery extends SQLObject {}

class SQLSelectQueryBlock implements SQLSelectQuery {
    List<SQLSelectItem> selectList;
    SQLTableSource from;
    SQLExpr where;
    SQLSelectGroupByClause groupBy;
    SQLOrderBy orderBy;
    SQLLimit limit;
}

class SQLUnionQuery implements SQLSelectQuery {
    SQLSelectQuery left;
    SQLSelectQuery right;
    SQLUnionOperator operator; // UNION/UNION_ALL/MINUS/INTERSECT
}

![复制代码](https://common.cnblogs.com/images/copycode.gif)

以下是SQLSelectQueryBlock中包含的主要节点

sql

ast

字段

SQLSelectItems

表

SQLTableSource

where条件

SQLExpr

groupby

SQLSelectGroupByClause

orderby

SQLOrderBy

limit

SQLLimit

表格中的这些ast节点都是SQL对应语法的一些表示，相信大家都非常熟悉，根据名字也轻易能了解具体是语法。

这里需要细化一下SQLTableSource这个节点，它有着常见的实现SQLExprTableSource（from的表）、SQLJoinTableSource（join的表）、SQLSubqueryTableSource（子查询的表）如：

![复制代码](https://common.cnblogs.com/images/copycode.gif)

class SQLTableSourceImpl extends SQLObjectImpl implements SQLTableSource { 
    String alias;
}

// 例如 select * from emp where i = 3，这里的from emp是一个SQLExprTableSource
// 其中expr是一个name=emp的SQLIdentifierExpr
class SQLExprTableSource extends SQLTableSourceImpl {
    SQLExpr expr;
}

// 例如 select * from emp e inner join org o on e.org_id = o.id
// 其中left 'emp e' 是一个SQLExprTableSource，right 'org o'也是一个SQLExprTableSource
// condition 'e.org_id = o.id'是一个SQLBinaryOpExpr
class SQLJoinTableSource extends SQLTableSourceImpl {
    SQLTableSource left;
    SQLTableSource right;
    JoinType joinType; // INNER_JOIN/CROSS_JOIN/LEFT_OUTER_JOIN/RIGHT_OUTER_JOIN/...
    SQLExpr condition;
}

// 例如 select * from (select * from temp) a，这里第一层from(...)是一个SQLSubqueryTableSource
SQLSubqueryTableSource extends SQLTableSourceImpl {
    SQLSelect select;
}

![复制代码](https://common.cnblogs.com/images/copycode.gif)

另外SQLExpr出现的地方也比较多，比如where语句，join条件，SQLSelectItem中等，因此，也需要细化了解一下，如

![复制代码](https://common.cnblogs.com/images/copycode.gif)

// SQLName是一种的SQLExpr的Expr，包括SQLIdentifierExpr、SQLPropertyExpr等
public interface SQLName extends SQLExpr {}

// 例如 ID = 3 这里的ID是一个SQLIdentifierExpr
class SQLIdentifierExpr implements SQLExpr, SQLName {
    String name;
} 

// 例如 A.ID = 3 这里的A.ID是一个SQLPropertyExpr
class SQLPropertyExpr implements SQLExpr, SQLName {
    SQLExpr owner;
    String name;
} 

// 例如 ID = 3 这是一个SQLBinaryOpExpr
// left是ID (SQLIdentifierExpr)
// right是3 (SQLIntegerExpr)
class SQLBinaryOpExpr implements SQLExpr {
    SQLExpr left;
    SQLExpr right;
    SQLBinaryOperator operator;
}

// 例如 select * from where id = ?，这里的?是一个SQLVariantRefExpr，name是'?'
class SQLVariantRefExpr extends SQLExprImpl { 
    String name;
}

// 例如 ID = 3 这里的3是一个SQLIntegerExpr
public class SQLIntegerExpr extends SQLNumericLiteralExpr implements SQLValuableExpr { 
    Number number;

    // 所有实现了SQLValuableExpr接口的SQLExpr都可以直接调用这个方法求值
    @Override
    public Object getValue() {
        return this.number;
    }
}

// 例如 NAME = 'jobs' 这里的'jobs'是一个SQLCharExpr
public class SQLCharExpr extends SQLTextLiteralExpr implements SQLValuableExpr{
    String text;
}

![复制代码](https://common.cnblogs.com/images/copycode.gif)

SqlParser定义了完整的ast各个节点对象，一条SQL语句被解析成这些对象的树形结构，而我们要做的就是根据这样的一个树形结构去做相应的处理。以上代码片段摘取了部分wiki上的，并调整了一下顺序，完整的wiki可以到druid的github上查阅。

## 使用示例

![复制代码](https://common.cnblogs.com/images/copycode.gif)

public void enhanceSql(String sql) {
        // 解析
        List<SQLStatement> statements = SQLUtils.parseStatements(sql, JdbcConstants.MYSQL);
        // 只考虑一条语句
        SQLStatement statement = statements.get(0);
        // 只考虑查询语句
        SQLSelectStatement sqlSelectStatement = (SQLSelectStatement) statement;
        SQLSelectQuery     sqlSelectQuery     = sqlSelectStatement.getSelect().getQuery();
        // 非union的查询语句
        if (sqlSelectQuery instanceof SQLSelectQueryBlock) {
            SQLSelectQueryBlock sqlSelectQueryBlock = (SQLSelectQueryBlock) sqlSelectQuery;
            // 获取字段列表
            List<SQLSelectItem> selectItems         = sqlSelectQueryBlock.getSelectList();
            selectItems.forEach(x -> {
                // 处理---------------------
            });
            // 获取表
            SQLTableSource table = sqlSelectQueryBlock.getFrom();
            // 普通单表
            if (table instanceof SQLExprTableSource) {
                // 处理---------------------
            // join多表
            } else if (table instanceof SQLJoinTableSource) {
                // 处理---------------------
            // 子查询作为表
            } else if (table instanceof SQLSubqueryTableSource) {
                // 处理---------------------
            }
            // 获取where条件
            SQLExpr where = sqlSelectQueryBlock.getWhere();
            // 如果是二元表达式
            if (where instanceof SQLBinaryOpExpr) {
                SQLBinaryOpExpr   sqlBinaryOpExpr = (SQLBinaryOpExpr) where;
                SQLExpr           left            = sqlBinaryOpExpr.getLeft();
                SQLBinaryOperator operator        = sqlBinaryOpExpr.getOperator();
                SQLExpr           right           = sqlBinaryOpExpr.getRight();
                // 处理---------------------
            // 如果是子查询
            } else if (where instanceof SQLInSubQueryExpr) {
                SQLInSubQueryExpr sqlInSubQueryExpr = (SQLInSubQueryExpr) where;
                // 处理---------------------
            }
            // 获取分组
            SQLSelectGroupByClause groupBy = sqlSelectQueryBlock.getGroupBy();
            // 处理---------------------
            // 获取排序
            SQLOrderBy orderBy = sqlSelectQueryBlock.getOrderBy();
            // 处理---------------------
            // 获取分页
            SQLLimit limit = sqlSelectQueryBlock.getLimit();
            // 处理---------------------
        // union的查询语句
        } else if (sqlSelectQuery instanceof SQLUnionQuery) {
            // 处理---------------------
        }
    }

![复制代码](https://common.cnblogs.com/images/copycode.gif)

以上示例中只是简单判断了一下类型，实际项目中你可能需要对整个ast做递归之类的方式来处理节点。其实当SQL语句变成了ast结构以后，我们只要知道这个ast结构存在什么样的节点，获取节点判断类型并做相应的操作即可，至于你是递归还是访问者模式还是别的什么方式去处理ast都可以。

SqlParser官方wiki

抽象语法树： [https://github.com/alibaba/druid/wiki/Druid_SQL_AST#24-sqlselect--sqlselectquery](https://github.com/alibaba/druid/wiki/Druid_SQL_AST#24-sqlselect--sqlselectquery)

解析器：[https://github.com/alibaba/druid/wiki/SQL-Parser](https://github.com/alibaba/druid/wiki/SQL-Parser)

visitor访问者示例：[https://github.com/alibaba/druid/wiki/SQL_Parser_Demo_visitor](https://github.com/alibaba/druid/wiki/SQL_Parser_Demo_visitor)