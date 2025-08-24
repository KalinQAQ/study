上一节我们快速入门了 GraphQL，为了让本小册更为完整，本节会继续介绍 GraphQL 的各个知识点。木木学长在学习 GraphQL 的过程是比较曲折的，国内也缺少比较好的 GraphQL 学习资料。因此我也一直思考如何让文章更清晰易读，而不是单纯的入个门而已，才能够更好的帮助到大家。同时也欢迎大家在评论区发表自己的观点。



# GraphQL

GraphQL 官网：https://graphql.org/

GraphQL 官网（中文）：https://graphql.cn/

GraphQL.js 官网：https://graphql.cn/graphql-js


## GraphQL.js

我们说 GraphQL 是一门查询语言，不依赖特定的编程语言。但是，很多语言都提供了 GraphQL 的实现，比如：

- JavaScript: https://github.com/graphql/graphql-js
- Go: https://github.com/graphql-go/graphql
- Java: https://github.com/graphql-java/graphql-java
- Python: https://github.com/graphql-python/graphene
- PHP: https://github.com/webonyx/graphql-php
- 等等...


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dcb172d1e00441598d086a09c4dced92~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2186&h=1236&s=145726&e=png&b=ffffff)

我们通过一个例子来演示如何在 JavaScript 中实现 API 查询。

首先我们需要安装 [graphql](https://www.npmjs.com/package/graphql) 这个库，graphql 最新版本是 16，相比 15 有一些 breaking change，但是官网对应文档没有更新，所以这里我们还是安装 15 版本：


```js
npm i graphql@15
```

然后我们就可以编写代码了：


```js
const { graphql, buildSchema } = require("graphql");

// 使用 GraphQL schema language 构建一个 schema
const schema = buildSchema(`
  type Query {
    hello: String
  }
`);

// 根节点为每个 API 入口端点提供一个 resolver 函数
const resolver = {
  hello: () => {
    return "Hello world!";
  },
};

const query = `
  query HelloWorld {
    hello
  }
`;

// 运行 GraphQL query '{ hello }' ，输出响应
graphql(schema, query, resolver).then((response) => {
  console.log(response);
});
```

最后我们运行 `node graphql.js`，可以看到 `Hello world!` 被正常的打印出来了：

```js
{ data: { hello: 'Hello world!' } }
```



## 查询语句

### 查询语句结构

<p align=center><img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b4427254015b4005ae5e115f25362e65~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1872&h=1456&s=181934&e=png&b=ffffff" alt="image.png" width="70%" /></p>

可以看到这里包含了几个概念：

- 操作类型（Operation Type）
- 操作名称（Operation Name）
- 变量（Variables）
- 服务器响应内容（Fields）

让我们来一一来解释这几个概念。

**操作类型**

GraphQL 有三种重要的操作类型，分别是：

- 查询（query）
- 变更（mutation）
- 订阅（subscription）

其中查询（query）和变更（mutation）比较简单我们上节课演示过了。那什么是订阅（subscription）呢？

在 GraphQL 中 subscription 使得客户端能够实时接收数据更新，适合实时聊天等需要实时接收数据变化的功能。在背后使用了 WebSocket，在后面 Apollo 部分我们会实现一个 subscription 的案例。

**操作名称**

操作名称是你的操作的有意义和明确的名称，虽然它是可选的，但我们鼓励使用它，因为它对于调试和服务器端日志记录非常有用。


**变量**

在很多场景中我们需要动态传递参数，比如当用户输入一本书的 `id`，我们能够根据该 `id` 返回对应书本的信息。在上面的查询语句中：


```graphql
query Book($bookId: ID)  {
  book(id: $bookId) {
    title,
    author {
      name
    }
  }
}
```

我们声明了 `$bookId` 变量，紧接着将 `$bookId` 作为参数传入到了 `book` 中。上面 `$bookId` 是一个可选传入，如果我们要求必传，则需要加上 `!` 


```graphql
query Book($bookId: ID!)  {
  book(id: $bookId) {
    title,
    author {
      name
    }
  }
}
```

我们也可以给变量赋一个默认值，如果未传入参数则采用默认值：


```graphql
query Book($bookId: ID = 0)  {
  book(id: $bookId) {
    title,
    author {
      name
    }
  }
}
```


**服务器响应内容**

在 GraphQL 中，服务器返回什么样的结果是由客户端决定的，例如当我们查询：


```graphql
{
  book {
    name
  }
}
```

得到的结果：


```json
{
  "data": {
    "book": {
      "name": "老人与海"
    }
  }
}
```

这样我们就能够通过 GraphQL 一次性拿到所有的数据，而不像 REST API 一样需要分多次返回，而且拿到的结果是可被预测的，也就是说我们能够知道返回数据的结构是怎么样的，这大大提高了我们前后端协作的效率。



### 片段（Fragments）

当我们拥有不同的查询语句，而这些查询中包含了一些共同的字段。例如我们很容易想到用户购物车和购物记录中查询语句中会包含大量的共同字段，此时我们就可以借助 Fragments 能力来实现：


```graphql
# 可复用单元
fragment cartItemDetails on Product {
  id 
  name
  price 
  description
}

query GetUserCart($userId: ID!) {
  userCart(userId: $userId) {
    items {
      ...cartItemDetails
    }
    total
  }
}

query GetUserPurchaseHistory($userId: ID!) {
  userPurchases(userId: $userId) {
    items {
      ...cartItemDetails
    }
    purchaseDate
  }
}

```

这样不仅简化了代码的实现，也使得维护更加简单。





## Schema

在 GraphQL 服务器中我们需要定义一套类型，我们称之为 Schema，用来描述可以从我们这个 GraphQL 服务中查到哪些数据，当查询过来的时候服务器就会根据 Schema 验证并执行查询。

例如我们在上一章中定义的 Schema：


```graphql
const typeDefs = `
  type Book {
    title: String
    authors: [Author]
    year: Int
  }

  type Author {
    name: String
    books: [Book]
    nationality: String
  }

  type Query {
    books: [Book]
    authors: [Author]
  }
`
```






**基本的数据类型**

在 GraphQL 中包含了以下几个基本的数据类型：

- `Int`：整数。
-   `Float`：小数。
-   `String`：字符串。
-   `Boolean`：`true` 或者 `false`。
-   `ID`：唯一标识符，比如对于每个用户来说，我们需要唯一标识用户身份信息的 ID，会作为 String 序列化给前端。

**枚举类型**

枚举类型限制了数据在可选的范围之内，例如：


```graphql
enum Log {
  warn
  error
  info
}

type Query {
  log: Log
}
```

这样当我们 Resolver 必须返回 `"warn"、"error"、"info"` 之一：


```js
const resolver = {
  log: () => {
    return "warn"; // 这里必须返回 "warn"、"error"、"info" 之一
  },
};

const query = `
  query LogLevel {
    log
  }
`;

graphql(schema, query, resolver).then((response) => {
  console.log(response);
});
```

如果返回其他值则会报错：`GraphQLError [Object]: Enum "Log" cannot represent value: "hello world"`

**非空**

如果我们希望是一个非空类型，我们可以在后面加上 `!`：


```graphql
type Query {
  log: Log!
}
```

如果返回是一个空，则会报错：


```js
const resolver = {
  log: () => {}, // Error: Cannot return null for non-nullable field Query.log.
};
```

**列表**

我们通过将类型包在方括号（`[` 和 `]`）中的方式来标记列表，例如：

```graphql
type Book {
  title: String
  authors: [Author]
  year: Int
}
```

中 `[Author]` 代表返回的 `Author` 类型的数组，即一本书对应了多个作者。

**接口**

跟许多类型系统一样，GraphQL 支持接口（interface）。一个接口包含了一些字段，而对象类型必须包含这些字段，才能算实现了这个接口。

比如有一个系统包含了不同类别的书，比如纸质版书籍和电子版书籍，而这些书籍包含了共同的属性，比如作者和标题，在此基础之上纸质版书籍和电子版书籍拥有不同的属性。


```graphql
interface Book {
  id: ID!
  title: String!
  author: String!
}

type PhysicalBook implements Book {
  id: ID!
  title: String!
  author: String!
  inventoryCount: Int!
}

type EBook implements Book {
  id: ID!
  title: String!
  author: String!
  downloadUrl: String!
}
```


**输入类型**

在前面我们演示了如何传入参数，但是有时候你需要传递一整个对象作为参数，这时候在 Schema 中就需要定义 `input`：


```graphql
input ReviewInput {
  stars: Int!
  commentary: String
}
```

然后在查询语句中将该 `input` 传入：


```graphql
mutation CreateReviewForEpisode($ep: Episode!, $review: ReviewInput!) {
  createReview(episode: $ep, review: $review) {
    stars
    commentary
  }
}
```

## Resolver




## Express 与 GraphQL

在前面我们演示了如何使用 GraphQL.js 编写一个 `Hello world!` 脚本。现在，我们使用 Express 将它变成一个服务器，这样可以来处理客户端的 GraphQL 查询请求。

首先安装相关的库：


```js
npm install express express-graphql graphql
```

然后编写 `server.js` 脚本：


```js
// server.js
const { buildSchema } = require('graphql')
const express = require('express')
const { graphqlHTTP } = require('express-graphql')

// 使用 GraphQL schema language 构建一个 schema
const schema = buildSchema(`
  type Query {
    hello: String
  }
`)

// 根节点为每个 API 入口端点提供一个 resolver 函数
const resolver = {
  hello: () => {
    return 'Hello world!'
  },
}

const app = express()

app.use(
  '/graphql',
  graphqlHTTP({
    schema: schema,
    rootValue: resolver,
    graphiql: true,
  }),
)

app.listen(4000)

console.log('Running a GraphQL API server at http://localhost:4000/graphql');
```

然后执行命令来启动服务器：


```js
node server.js
```

由于我们上面传入了 `graphiql: true`，因此我们可以使用 [GraphiQL](https://github.com/graphql/graphiql/tree/main/packages/graphiql) 工具来手动执行 GraphQL 查询，在浏览器中打开 `http://localhost:4000/graphql`：



![20240810160152_rec_.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07e8c950a003456381da5fcbb8c1aa65~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1920&h=1080&s=254380&e=gif&f=137&b=f9f8fc)




# 总结

本章全面的介绍了 GraphQL 的语法，以及 GraphQL.js 库的使用。在接下来的章节我们会介绍 Apollo 以及 NestJS 的使用，并对 GraphQL 相关的 NodeJS 生态进行总结。