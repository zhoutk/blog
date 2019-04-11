# Graphql实战系列（一）

## 背景介绍
  graphql越来越流行，一直想把我的凝胶项目除了支持restful api外，也能同时支持graphql。由于该项目的特点是结合关系数据库的优点，尽量少写重复或雷同的代码。对于rest api，在做完数据库设计后，百分之六十到八十的接口就已经完成了，但还需要配置上api文档。而基于数据库表自动实现graphql，感觉还是有难度的，但若能做好，连文档也就同时提供了。  
    
  不久前又看到了一句让我深以为然的话：No program is perfect, even the most talented engineers will write a bug or two (or three). By far the best design pattern available is simply writing less code. That’s the opportunity we have today, to accomplish our goals by doing less.
