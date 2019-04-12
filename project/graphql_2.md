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

#### 
