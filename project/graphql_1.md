# Graphql实战系列（上）

## 背景介绍
  graphql越来越流行，一直想把我的凝胶项目除了支持restful api外，也能同时支持graphql。由于该项目的特点是结合关系数据库的优点，尽量少写重复或雷同的代码。对于rest api，在做完数据库设计后，百分之六十到八十的接口就已经完成了，但还需要配置上api文档。而基于数据库表自动实现graphql，感觉还是有难度的，但若能做好，连文档也就同时提供了。  
    
  不久前又看到了一句让我深以为然的话：No program is perfect, even the most talented engineers will write a bug or two (or three). By far the best design pattern available is simply writing less code. That’s the opportunity we have today, to accomplish our goals by doing less.  
  
so, ready go...

## 基本需求与约定
- 根据数据库表自动生成schema
- 充分利用已经有的支持restful api的底层接口
- 能自动实现一对多的表关系
- 能方便的增加特殊业务，只需要象rest一样，只需在指定目录，增加业务模块即可
- 测试表有两个，book & author
- book表字段有：id, title, author_id
- author表字段有： id, name
- 数据表必须有字段id，类型为整数(自增)或8位字符串(uuid)，作为主键或建立unique索引
- 表名为小写字母，使用名词单数，以下划作为单词分隔
- 表关联自动在相关中嵌入相关对象，Book对象增加Author对象，Author对象增加books列表
- 每个表会默认生成两个query，一个是以id为参数进行单条查询，另一个是列表查询；命名规则；单条查询与表名相同，列表查询为表名+s，若表名本身以s结尾，则变s为z

## 桥接库比较与选择
我需要在koa2上接入graphql，经过查阅资料，最后聚焦在下面两个库上：
- kao-graphql
- apollo-server-koa

## kao-graphql实现

开始是考虑简单为上，试着用kao-graphql，作为中间件可以方便的接入，我指定了/gql路由，可以测试效果，代码如下：
```
import * as Router from 'koa-router'
import BaseDao from '../db/baseDao'
import { GraphQLString, GraphQLObjectType, GraphQLSchema, GraphQLList, GraphQLInt } from 'graphql'
const graphqlHTTP = require('koa-graphql')
let router = new Router()

export default (() => {
    let authorType = new GraphQLObjectType({
        name: 'Author',
        fields: {
            id: { type: GraphQLInt},
            name: { type: GraphQLString}
        }
    })
    let bookType = new GraphQLObjectType({
        name: 'Book',
        fields: {
            id: { type: GraphQLInt},
            title: { type: GraphQLString},
            author: { 
                type: authorType,
                resolve: async (book, args) => {
                    let rs = await new BaseDao('author').retrieve({id: book.author_id})
                    return rs.data[0]
                }
            }
        }
    })
    let queryType = new GraphQLObjectType({
        name: 'Query',
        fields: {
            books: {
                type: new GraphQLList(bookType),
                args: {
                    id: { type: GraphQLString },
                    search: { type: GraphQLString },
                    title: { type: GraphQLString },
                },
                resolve: async function (_, args) {
                    let rs = await new BaseDao('book').retrieve(args)
                    return rs.data
                }
            },
            authors: {
                type: new GraphQLList(authorType),
                args: {
                    id: { type: GraphQLString },
                    search: { type: GraphQLString },
                    name: { type: GraphQLString },
                },
                resolve: async function (_, args) {
                    let rs = await new BaseDao('author').retrieve(args)
                    return rs.data
                }
            }
        }
    })

    let schema = new GraphQLSchema({ query: queryType })
    return router.all('/gql', graphqlHTTP({
        schema: schema,
        graphiql: true
    }))
})() 
```
> 这种方式有个问题，前面的变量对象中要引入后面定义的变量对象会出问题，因此投入了apollo-server。但apollo-server 2.0网上资料少，大多是介绍1.0的，而2.0变动又比较大，因此折腾了一段时间，还是要多看英文资料。
> apollo-server 2.0集成很多东西到里面，包括cors，bodyParse，graphql-tools 等。

## apollo-server 2.0实现静态schema

通过中间件加载，放到rest路由之前，加入顺序及方式请看app.ts，apollo-server-kao接入代码：
```
//自动生成数据库表的基础schema，并合并了手写的业务模块
import { getInfoFromSql } from './schema_generate'
const { ApolloServer } = require('apollo-server-koa')

export default async (app) => {    //app是koa实例
  let { typeDefs, resolvers } = await getInfoFromSql()  //数据库查询是异步的，所以导出的是promise函数
  if (!G.ApolloServer) {
    G.ApolloServer = new ApolloServer({
      typeDefs,                   //已经不需要graphql-tools，ApolloServer构造函数已经集成其功能
      resolvers,
      context: ({ ctx }) => ({    //传递ctx等信息，主要供认证、授权使用
        ...ctx,
        ...app.context
      })
    })
  }
  G.ApolloServer.applyMiddleware({ app })
}
```

静态schema试验，schema_generate.ts

```
const typeDefs = `
  type Author {
    id: Int!
    name: String
    books: [book]
  }
  type Book {
    id: Int!
    title: String
    author: Author
  }
  # the schema allows the following query:
  type Query {
    books: [Post]
    author(id: Int!): Author
  }
`

const resolvers = {
  Query: {
    books: async function (_, args) {
               let rs = await new BaseDao('book').retrieve(args)
               return rs.data
           },
    author: async function (_, { id }) {
               let rs = await new BaseDao('author').retrieve({id})
               return rs.data[0]
           },
  },
  Author: {
    books: async function (author) {
               let rs = await new BaseDao('book').retrieve({ author_id: author.id })
               return rs.data
           },
   },
   Book: {
    author: async function (book) {
               let rs = await new BaseDao('author').retrieve({ id: book.author_id })
               return rs.data[0]
           },
   },
}

export {
  typeDefs,
  resolvers
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
这是第一部分，确定需求，进行了技术选型，实现了接入静态手写schema试验，下篇将实现动态生成与合并特殊业务模型。
