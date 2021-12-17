---
layout: post
title: "nodejs中graphql的服务端和客户端实现"
subtitle: ""
author: ""
header-style: text
tags:
  - nodejs
  - graphql
---



先简单介绍一下GraphQL。

GraphQL 是一门出自Facebook，用于api的查询语言，被称作是Restful的替代品，已经有越来越多的公司和系统使用GraphQL来代替Restful。

它的几个主要特点是：

1. 只返回你想要的数据。传统Restful中返回的是对象的所有字段，而往往我们需要的只是其中几个字段，这样无疑造成了很大的带宽浪费。GraphQL中由你定义你的查询请求，想要什么就只给你返回什么，所以GraphQL也被称作是由前端自由定义的查询语言。
2. 一次请求中就可以获取多个资源，被称作数据聚合。传统Restful中经常会先发起一个请求获取某个资源，再根据这个资源的某个字段请求其他资源，最后由前端聚合成最终想要的数据。而GraphQL在一次请求中就可以拿到所有你想要的资源，并且遵循第一个特点，只包括你想要的字段。

由于如今前后端分离越来越普遍，前端对于api的要求也越来越灵活和复杂，GraphQL正是下一代的API技术。

 

服务端实现

这里采用了Nest的Node.js框架。在nest中实现GraphQL服务端非常方便。它的官方文档也写得非常清楚。这里不再作说明。推荐采用代码优先的方式，我们只需使用它提供的各种装饰器即可生成相应的 GraphQL 架构。说白了就是各种注解。

 

客户端

这是本篇文章的重点。包括 GraphQL 的 **Query、Mutations、Subscriptions** 的 3 种操作。

客户端使用的是 Apollo Client。它和各种UI框架作了很好的集成，如React 和 Vue ，可以直接将查询和变更操作绑定到组件上。

但我们也可以在 nodejs 环境种单独使用，它其中的 Apollo-Link 模块就提供了这种方式。



1、下面是针对 Query、Mutations 的示例代码：

```javascript
const { execute, makePromise } = require('apollo-link');
const { HttpLink } = require('apollo-link-http');
const gql = require('graphql-tag');
const fetch = require('node-fetch');

const query = gql`query { photos {id,name,isPublished}}`;

const mutation = gql`mutation($input:NewPhotoInput!) {
    addPhoto(newPhotoData:$input) {
        id,
        name,
        isPublished
    }
}`;

const variables = {
  'input': {
    'name': '安东尼',
    'description': '瓜哥',
  },
};

const uri = 'http://localhost:3000/graphql';
const link = new HttpLink({ uri, fetch });
const operation = {
  query: mutation,
  variables, //optional
  // operationName: {}, //optional
  // context: {}, //optional
  // extensions: {}, //optional
};

// execute returns an Observable so it can be subscribed to
execute(link, operation).subscribe({
  next: res => console.log(res.data),
  error: error => console.log(`received error ${error}`),
  complete: () => console.log(`complete`),
});

// For single execution operations, a Promise can be used
// makePromise(execute(link, operation))
//   .then(res => console.log(res.data))
//   .catch(error => console.log(`received error ${error}`));
```

[具体细节参考官方文档](https://www.apollographql.com/docs/link/#standalone) 

需要注意的一点是需要安装 `node-fetch`，在创建HttpLink的时候传递到它的选项对象中去。

execute 和 makePromise 方法选一个就行。

 

2、对于Subscriptions 操作就不能使用上面的 HttpLink 了。因为是订阅操作嘛，所以得使用 Websocket 连接。

```javascript
const { WebSocketLink } = require('apollo-link-ws');
const ws = require('ws');
const { SubscriptionClient } = require('subscriptions-transport-ws');
const { execute, makePromise } = require('apollo-link');
const gql = require('graphql-tag');


const GRAPHQL_ENDPOINT = 'ws://localhost:3000/graphql';

const client = new SubscriptionClient(GRAPHQL_ENDPOINT, {
  reconnect: true,
}, ws);

const link = new WebSocketLink(client);
const operation = {
  query: gql`subscription {photoAdded {id,name,isPublished}}`,
};

execute(link, operation).subscribe({
  next: res => console.log(res.data),
});
```

不同之处在于使用的是 subscriptions-transport-ws 模块中的 `SubscriptionClient`  作为客户端，表示建立的是 Websocket 连接。

另外建立的连接也不是 HttpLink，而是 apollo-link-ws 中的 `WebSocketLink` 。

[具体细节参考](https://github.com/apollographql/apollo-link/tree/master/packages/apollo-link-ws)