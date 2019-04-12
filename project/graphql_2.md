## Graphql实战系列（下）

## 前情介绍
在[《Graphql实战系列（上）》](https://segmentfault.com/a/1190000018837788)中我们已经完成技术选型，并将graphql桥接到[凝胶gels](https://github.com/zhoutk/gels)项目中，并动态手写了schema，可以通过 http://localhost:5000/graphql 查看效果。这一节，我们根据数据库表来自动生成基本的查询与更新schema，并能方便的扩展schema，实现我们想来的业务逻辑。

## 设计思路
对象定义在apollo-server中是用字符串来做的，而Query与Mutation只能有一个，而我们的定义又会分散在多个文件中，因此只能先以一定的形式把它们存入数组中，在生成schema前一刻再组合。

### 业务逻辑模块模板设计：
```
const customDefs = {
    textDefs: `
        type ReviseResult {
            id: Int
            affectedRows: Int
            status: Int
            message: String
        },
    queryDefs: [],
    mutationDefs: []
}

const customResolvers = {
    Query: {
    },
    Mutation: {
    }
 }
export { customDefs, customResolvers }
```

### schema合并算法

```
let typeDefs = []
    let dirGraphql = requireDir('../../graphql')		//从手写schema业务模块目录读入文件
    G.L.each(dirGraphql, (item, name) => {
        if (item && item.customDefs && item.customResolvers) {
            typeDefs.push(item.customDefs.textDefs || '')				//合并文本对象定义
            typeDefObj.query = typeDefObj.query.concat(item.customDefs.queryDefs || [])		//合并Query
            typeDefObj.mutation = typeDefObj.mutation.concat(item.customDefs.mutationDefs || [])  //合并Matation
            let { Query, Mutation, ...Other } = item.customResolvers
            Object.assign(resolvers.Query, Query)			//合并resolvers.Query
            Object.assign(resolvers.Mutation, Mutation)		//合并resolvers.Mutation
            Object.assign(resolvers, Other)					//合并其它resolvers
        }
    })
	//将query与matation查询更新对象由自定义的数组转化成为文本形式
    typeDefs.push(Object.entries(typeDefObj).reduce((total, cur) => {
        return total += `
            type ${G.tools.bigCamelCase(cur[0])} {
                ${cur[1].join('')}
            }
        `
    }, ''))
```

### 从数据库表动态生成schema
自动生成内容：
- 一个表一个对象；
- 每个表有两个Query，一是单条查询，二是列表查询；
- 三个Mutation，一是新增，二是更新，三是删除；
- 关联表以上篇中的Book与Author为例，Book中有author_id，会生成一个Author对象；而Author表中会生成一个对象列表[Book]

#### mysql类型 => graphql 类型转化常量定义
定义一类型转换，不在定义中的默认为String。
```
const TYPEFROMMYSQLTOGRAPHQL = {
    int: 'Int',
    smallint: 'Int',
    tinyint: 'Int',
    bigint: 'Int',
    double: 'Float',
    float: 'Float',
    decimal: 'Float',
}
```

#### 从数据库中读取数据表信息
```
    let dao = new BaseDao()
    let tables = await dao.querySql('select TABLE_NAME,TABLE_COMMENT from information_schema.`TABLES` ' +
        ' where TABLE_SCHEMA = ? and TABLE_TYPE = ? and substr(TABLE_NAME,1,2) <> ? order by ?',
        [G.CONFIGS.dbconfig.db_name, 'BASE TABLE', 't_', 'TABLE_NAME'])
```
#### 从数据库中读取表字段信息
```
tables.data.forEach((table) => {
        columnRs.push(dao.querySql('SELECT	`COLUMNS`.COLUMN_NAME,`COLUMNS`.COLUMN_TYPE,`COLUMNS`.IS_NULLABLE,' +
            '`COLUMNS`.CHARACTER_SET_NAME,`COLUMNS`.COLUMN_DEFAULT,`COLUMNS`.EXTRA,' +
            '`COLUMNS`.COLUMN_KEY,`COLUMNS`.COLUMN_COMMENT,`STATISTICS`.TABLE_NAME,' +
            '`STATISTICS`.INDEX_NAME,`STATISTICS`.SEQ_IN_INDEX,`STATISTICS`.NON_UNIQUE,' +
            '`COLUMNS`.COLLATION_NAME ' +
            'FROM information_schema.`COLUMNS` ' +
            'LEFT JOIN information_schema.`STATISTICS` ON ' +
            'information_schema.`COLUMNS`.TABLE_NAME = `STATISTICS`.TABLE_NAME ' +
            'AND information_schema.`COLUMNS`.COLUMN_NAME = information_schema.`STATISTICS`.COLUMN_NAME ' +
            'AND information_schema.`STATISTICS`.table_schema = ? ' +
            'where information_schema.`COLUMNS`.TABLE_NAME = ? and `COLUMNS`.table_schema = ?',
            [G.CONFIGS.dbconfig.db_name, table.TABLE_NAME, G.CONFIGS.dbconfig.db_name]))
    })
```
#### 几个工具函数
取数据库表字段类型，去除圆括号与长度信息
```
	getStartTillBracket(str: string) {
        return str.indexOf('(') > -1 ? str.substr(0, str.indexOf('(')) : str
    }
```
下划线分隔的表字段转化为big camel-case
```
    bigCamelCase(str: string) {
        return str.split('_').map((al) => {
            if (al.length > 0) {
                return al.substr(0, 1).toUpperCase() + al.substr(1).toLowerCase()
            }
            return al
        }).join('')
    }
```
下划线分隔的表字段转化为small camel-case
```
    smallCamelCase(str: string) {
        let strs = str.split('_')
        if (strs.length < 2) {
            return str
        } else {
            let tail = strs.slice(1).map((al) => {
                if (al.length > 0) {
                    return al.substr(0, 1).toUpperCase() + al.substr(1).toLowerCase()
                }
                return al
            }).join('')
            return strs[0] + tail
        }
    }
```
#### 字段是否以_id结尾，是表关联的标志
不以_id结尾，是正常字段,判断是否为null，处理必填
```
typeDefObj[table].unshift(`${col['COLUMN_NAME']}: ${typeStr}${col['IS_NULLABLE'] === 'NO' ? '!' : ''}\n`)
```
以_id结尾，则需要处理关联关系
```
	//Book表以author_id关联单个Author实体
	typeDefObj[table].unshift(`"""关联的实体"""
        ${G.L.trimEnd(col['COLUMN_NAME'], '_id')}: ${G.tools.bigCamelCase(G.L.trimEnd(col['COLUMN_NAME'], '_id'))}`)
    resolvers[G.tools.bigCamelCase(table)] = {
        [G.L.trimEnd(col['COLUMN_NAME'], '_id')]: async (element) => {
            let rs = await new BaseDao(G.L.trimEnd(col['COLUMN_NAME'], '_id')).retrieve({ id: element[col['COLUMN_NAME']] })
            return rs.data[0]
        }
    }
	//Author表关联Book列表
    let fTable = G.L.trimEnd(col['COLUMN_NAME'], '_id')
    if (!typeDefObj[fTable]) {
        typeDefObj[fTable] = []
    }
    if (typeDefObj[fTable].length >= 2)
        typeDefObj[fTable].splice(typeDefObj[fTable].length - 2, 0, `"""关联实体集合"""${table}s: [${G.tools.bigCamelCase(table)}]\n`)
    else
        typeDefObj[fTable].push(`${table}s: [${G.tools.bigCamelCase(table)}]\n`)
    resolvers[G.tools.bigCamelCase(fTable)] = {
        [`${table}s`]: async (element) => {
            let rs = await new BaseDao(table).retrieve({ [col['COLUMN_NAME']]: element.id})
            return rs.data
        }
    }
```
#### 生成Query查询
单条查询
```
	if (paramId.length > 0) {
        typeDefObj['query'].push(`${G.tools.smallCamelCase(table)}(${paramId}!): ${G.tools.bigCamelCase(table)}\n`)
        resolvers.Query[`${G.tools.smallCamelCase(table)}`] = async (_, { id }) => {
            let rs = await new BaseDao(table).retrieve({ id })
            return rs.data[0]
        }
    } else {
        G.logger.error(`Table [${table}] must have id field.`)
    }
```
列表查询
```
    let complex = table.endsWith('s') ? (table.substr(0, table.length - 1) + 'z') : (table + 's')
    typeDefObj['query'].push(`${G.tools.smallCamelCase(complex)}(${paramStr.join(', ')}): [${G.tools.bigCamelCase(table)}]\n`)
    resolvers.Query[`${G.tools.smallCamelCase(complex)}`] = async (_, args) => {
        let rs = await new BaseDao(table).retrieve(args)
        return rs.data
    }
```
#### 生成Mutation查询
```
	typeDefObj['mutation'].push(`
            create${G.tools.bigCamelCase(table)}(${paramForMutation.slice(1).join(', ')}):ReviseResult
            update${G.tools.bigCamelCase(table)}(${paramForMutation.join(', ')}):ReviseResult
            delete${G.tools.bigCamelCase(table)}(${paramId}!):ReviseResult
        `)
    resolvers.Mutation[`create${G.tools.bigCamelCase(table)}`] = async (_, args) => {
        let rs = await new BaseDao(table).create(args)
        return rs
    }
    resolvers.Mutation[`update${G.tools.bigCamelCase(table)}`] = async (_, args) => {
        let rs = await new BaseDao(table).update(args)
        return rs
    }
    resolvers.Mutation[`delete${G.tools.bigCamelCase(table)}`] = async (_, { id }) => {
        let rs = await new BaseDao(table).delete({ id })
        return rs
    }
```
## 项目地址
```
https://github.com/zhoutk/gels
```
## 使用方法
```
git clone https://github.com/zhoutk/gels
cd gels
yarn
tsc -w
nodemon dist/index.js
```
然后就可以用浏览器打开链接：http://localhost:5000/graphql 查看效果了。

## 小结
我只能把大概思路写出来，让大家有个整体的概念，若想很好的理解，得自己把项目跑起来，根据我提供的思想，慢慢的去理解。因为我在编写的过程中还是遇到了不少的难点，这块既要自动化，还要能方便的接受手动编写的schema模块，的确有点难度。
