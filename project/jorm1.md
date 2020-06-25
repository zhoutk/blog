## c++操作关系数据库通用接口设计（JSON-ORM c++版）

重操C++旧业，习惯通常的数据库操作方式，因此抽时间，把JSON-ORM封装了一个C++版，现支持sqlit3与mysql，postgres已经做好了准备。  

### 设计思路
  我们通用的ORM，基本模式都是想要脱离数据库的，几乎都在编程语言层面建立模型，由程序去与数据库打交道。虽然脱离了数据库的具体操作，但我们要建立各种模型文档，用代码去写表之间的关系等等操作，让初学者一时如坠云雾。我的想法是，将关系数据库拥有的完善设计工具之优势，来实现数据设计以提供结构信息，让json对象自动映射成为标准的SQL查询语句。只要我们理解了标准的SQL语言，我们就能够完成数据库查询操作。

### 技术选择
#### json库
  JSON-ORM 数据传递采用json来实现，使数据标准能从最前端到最后端进行和谐统一。我选择了rapidjson，但rapidjson为了执行效率，直接操作内存，并且大量使用std::move，在使用的时候有很多限制，一不小心还会出现内存访问冲突。 因此我封装了它，提供了一个容易操作的Rjson代理类。
    
#### 数据库通用接口
  > 应用类直接操作这个通用接口，实现与底层实现数据库的分离。该接口提供了CURD标准访问，以及批量插入和事务操作，基本能满足平时百分之九十以上的数据库操作。

  ```
    class Idb
    {
    public:
      virtual Rjson select(string tablename, Rjson& params, vector<string> fields = vector<string>(), int queryType = 1) = 0;
      virtual Rjson create(string tablename, Rjson& params) = 0;
      virtual Rjson update(string tablename, Rjson& params) = 0;
      virtual Rjson remove(string tablename, Rjson& params) = 0;
      virtual Rjson querySql(string sql, Rjson& params = Rjson(), vector<string> filelds = vector<string>()) = 0;
      virtual Rjson execSql(string sql) = 0;
      virtual Rjson insertBatch(string tablename, vector<Rjson> elements) = 0;
      virtual Rjson transGo(vector<string> sqls, bool isAsync = false) = 0;
    };
  ```
    
#### 底层数据库访问实现
  现在已经实现了sqlit3与mysql的所有功能，postgres与oracle也做了技术准备，应该在不久的将来会实现（取决于是否有时间），mssql除非有特别的需求，短期内不会去写。  
  我选择的技术实现方式，基本上是最底层高效的方式。sqlit3 - sqllit3.h（官方的标准c接口）；mysql - c api （MySQL Connector C 6.1）；postgres - pqxx；oracle - occi；mssql - ？
    
#### 智能查询方式设计
> 查询保留字：page, size, sort, fuzzy, lks, ins, ors, count, sum, group

- page, size, sort, 分页排序
    在sqlit3与mysql中这比较好实现，limit来分页是很方便的，排序只需将参数直接拼接到order by后就好了。  
    查询示例：
    ```
    Rjson p;
    p.AddValueInt("page", 1);
    p.AddValueInt("size", 10);
    p.AddValueString("size", "sort desc");
    (new BaseDb(...))->select("users", p);
    
    生成sql：   SELECT * FROM users  ORDER BY age desc LIMIT 0,10
    ```
- fuzzy, 模糊查询切换参数，不提供时为精确匹配
    提供字段查询的精确匹配与模糊匹配的切换。
    ```
    Rjson p;
    p.AddValueString("username", "john");
    p.AddValueString("password", "123");
    p.AddValueString("fuzzy", "1");
    (new BaseDb(...))->select("users", p);
   
    生成sql：   SELECT * FROM users  WHERE username like '%john%'  and password like '%123%'
- ins, lks, ors
    这是最重要的三种查询方式，如何找出它们之间的共同点，减少冗余代码是关键。

    - ins, 数据库表单字段in查询，一字段对多个值，例：  
        查询示例：
        ```
        Rjson p;
        p.AddValueString("ins", "age,11,22,36");
        (new BaseDb(...))->select("users", p);

        生成sql：   SELECT * FROM users  WHERE age in ( 11,22,26 )
        ```
    - ors, 数据库表多字段精确查询，or连接，多个字段对多个值，例：  
        查询示例：
        ```
        Rjson p;
        p.AddValueString("ors", "age,11,age,36");
        (new BaseDb(...))->select("users", p);

        生成sql：   SELECT * FROM users  WHERE  ( age = 11  or age = 26 )
        ```
    - lks, 数据库表多字段模糊查询，or连接，多个字段对多个值，例：
        查询示例：
        ```
        Rjson p;
        p.AddValueString("lks", "username,john,password,123");
        (new BaseDb(...))->select("users", p);

        生成sql：   SELECT * FROM users  WHERE  ( username like '%john%'  or password like '%123%'  )
        ```
- count, sum
    这两个统计求和，处理方式也类似，查询时一般要配合group与fields使用。
    - count, 数据库查询函数count，行统计，例：
        查询示例：
        ```
        Rjson p;
        p.AddValueString("count", "1,total");
        (new BaseDb(...))->select("users", p);

        生成sql：   SELECT *,count(1) as total  FROM users
        ```
    - sum, 数据库查询函数sum，字段求和，例：
        查询示例：
        ```
        Rjson p;
        p.AddValueString("sum", "age,ageSum");
        (new BaseDb(...))->select("users", p);

        生成sql：   SELECT username,sum(age) as ageSum  FROM users
    ```
- group, 数据库分组函数group，例：  
    查询示例：
    ```
    Rjson p;
    p.AddValueString("group", "age");
    (new BaseDb(...))->select("users", p);

    生成sql：   SELECT * FROM users  GROUP BY age
    ```

> 不等操作符查询支持

支持的不等操作符有：>, >=, <, <=, <>, =；逗号符为分隔符，一个字段支持一或二个操作。  
特殊处：使用"="可以使某个字段跳过search影响，让模糊匹配与精确匹配同时出现在一个查询语句中

- 一个字段一个操作，示例：
    查询示例：
    ```
    Rjson p;
    p.AddValueString("age", ">,10");
    (new BaseDb(...))->select("users", p);

    生成sql：   SELECT * FROM users  WHERE age> 10
    ```
- 一个字段二个操作，示例：
    查询示例：
    ```
    Rjson p;
    p.AddValueString("age", ">=,10,<=,33");
    (new BaseDb(...))->select("users", p);

    生成sql：   SELECT * FROM users  WHERE age>= 10 and age<= 33
    ```
- 使用"="去除字段的search影响，示例：
    查询示例：
    ```
    Rjson p;
    p.AddValueString("age", "=,18");
    p.AddValueString("username", "john");
    p.AddValueString("fuzzy", "1");
    (new BaseDb(...))->select("users", p);

    生成sql：   SELECT * FROM users  WHERE age= 18  and username like '%john%'
    ```
    
#### 系列文章规划
  - c++操作关系数据库通用接口设计（JSON-ORM c++版）
  - Rjson -- rapidjson代理的设计与实现
  - sqlit3 数据库操作的实现与解析
  - mysql 数据库操作的实现与解析
  - postgres 数据库操作的实现与解析
  - oracle 数据库操作的实现与解析
  - mssql 数据库操作的实现与解析
  - 总结（如果需要的话）
  
#### 项目地址
```
https://github.com/zhoutk/Jorm
```

#### 感受
  多年没写C++了，恢复了一段时间，了解了新的C++标准，感觉世界变化真快。回顾这些年，我选择学习技术比较前卫，这些年主做node.js（Typescript），偶尔写python, go。已经习惯了动态语言，函数式编程。突然回归C++，发现好多普通的想法，实现起来还真麻烦。这就是人们的选择吧，有得就有失。想当年我是疯狂的迷恋C、C++，因此它们不限制我选择编程范式。这些年，随着硬件技术的发展，软件业的思维也突飞猛进，一般情况下不再为内存，算法效率而忧心，我们更注重开发效率和程序代码的人性化。回到C++的怀抱，看着我半墙的C++书箱，熟悉新c++标准，我选定了主攻方向，模板......
