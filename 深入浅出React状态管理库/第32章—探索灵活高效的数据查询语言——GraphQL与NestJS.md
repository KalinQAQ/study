本节我们来介绍一下在 Nest 中如何集成 GraphQL，我们前面说 GraphQL 有三个核心能力，即 Query、Mutation、Subscription，本节会重点介绍 Query 与 Mutation，在后续实战项目中我们会演示 Subscription 结合 Nest 的使用。

在阅读本章之前，需要你了解 Nest 的基本使用。
Nest 与 GraphQL 相关内容参见官网：https://docs.nestjs.com/graphql/quick-start



# 从一个案例开始

按照惯例，我们还是从一个案例开始，演示如何在 Nest 中如何集成 GraphQL。在这个案例中，我们会实现一个书本、作者的查询以及更新功能。
代码参见：https://github.com/L-Qun/state-management-collection/tree/main/examples/graphql/nestjs

## 项目启动

首先我们需要创建一个新的项目，我们借助 Nest CLI 来生成：


```js
npx @nestjs/cli new nestjs
```

这里的 `nestjs` 代表项目名。


然后我们需要安装相关的包：


```js
npm i @nestjs/graphql @nestjs/apollo @apollo/server graphql
```

安装包后，我们可以导入 `GraphQLModule` 并使用 `forRoot()` 静态方法进行配置。

```ts
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
    }),
  ],
})
export class AppModule {}
```

可以看到，我们安装了 `@apollo/server`，并传入了 `ApolloDriver`，如果我们想使用别的包，例如 Mercurius，我们需要对应安装 Mercurius 相关的依赖：


```js
npm i @nestjs/graphql @nestjs/mercurius graphql mercurius
```

并传入 `MercuriusDriver`：

```ts
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { MercuriusDriver, MercuriusDriverConfig } from '@nestjs/mercurius';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusDriverConfig>({
      driver: MercuriusDriver,
    }),
  ],
})
export class AppModule {}
```


也就是说 Nest 只是提供了一个公共的集成，让 GraphQL 与 Nest 结合更简单，背后还是使用的 `@apollo/server` 或者 `mercurius`。

和前面章节演示的例子一样，我们需要配置 Schema 和 Resolver。分别用来定义具体类型和请求数据。

## Schema 定义



```graphql
# src/books/books.graphql

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

type Mutation {
  addBook(title: String!, year: Int!, authorNames: [String]!): Book
}
```

## Resolver

```ts
// books.resolver.ts

import { Args, Mutation, Query, Resolver } from '@nestjs/graphql';
import { authors, books } from './mock';

@Resolver()
export class BooksResolver {
  @Query()
  books() {
    return books;
  }

  @Query('authors')
  getAuthors() {
    return authors;
  }

  @Mutation()
  addBook(
    @Args('title') title: string,
    @Args('year') year: number,
    @Args('authorNames') authorNames: string[],
  ) {
    const newBook = {
      title,
      year,
      authors: authors.filter((author) => authorNames.includes(author.name)),
    };
    books.push(newBook);
    return newBook;
  }
}
```

> 所有的装饰器，比如这里的 @Resolver、@Query、@Mutation、@Args 等等都从 @nestjs/graphql 导出

可以看到如果在 `@Query()` 中不加参数，则会使用方法名对应 Schema，例如上面的 `books` 方法对应了 Schema 中的 `books`，上面的 `@Query('authors')` 对应 Schema 中的 `authors`。

然后我们就可以在 `localhost:3000/graphql` 中看到调试工具：


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed078624d4824ad585af0be7941c438c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=3456&h=1984&s=230582&e=png&b=0e1e2a)

我们称它为 Playground。但是 Apollo Sandbox 会更好用一些，如果你希望还是使用 Apollo Sandbox，需要额外配置 `playground: false` 以及增加 `plugins`：


```ts
// app.module.ts

import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloServerPluginLandingPageLocalDefault } from '@apollo/server/plugin/landingPage/default';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      playground: false,
      plugins: [ApolloServerPluginLandingPageLocalDefault()],
    }),
  ],
})
export class AppModule {}
```

这样我们可以使用 Apollo Sandbox 来进行调试了！


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb6b3f83473146b2af982d1186a03982~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=3456&h=1984&s=233866&e=png&b=f2f4f4)


最后我们需要将 `BooksResolver` 加入到 `providers` 中：


```ts
@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      playground: false,
      plugins: [ApolloServerPluginLandingPageLocalDefault()],
      typePaths: ['./**/*.graphql'],
      definitions: {
        path: join(process.cwd(), 'src/graphql.ts'),
      },
    }),
  ],
+ providers: [BooksResolver],
})
export class AppModule {}
```

我们在这里只定义了 `BooksResolver`，也可以定义多个 Resolver 类，Nest 会在运行时将它们组合起来。


## 类型生成

试想一下上面我们的例子有什么问题？

我们需要重复定义两遍类型，即相当于在 GraphQL Schema 中定义了一次类型，在 TS 中又重复定义了一遍类型，这增大了编写代码的负担也带来了出错的风险。

是否有一种方式可以自动根据 Schema 定义来为我们生成 TS 类型呢？

我们可以在 `GraphQLModule` 时添加 `definitions` 属性：


```ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
  definitions: {
    path: join(process.cwd(), 'src/graphql.ts'),
  },
}),
```

这样 Nest 就会自动为我们在指定的路径下，即 `src/graphql.ts` 文件中生成正确的类型定义：


<p align=center><img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f61a62a8f1c8484d818881e9a655cfbc~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1900&h=1500&s=315716&e=png&b=282c34" alt="image.png" width="80%" /></p>

背后 Nest 在**每次应用启动时**会根据 AST 抽象语法树自动生成 TypeScript 类型。

通常我们需要当 `*.graphql` 文件更新时自动生成类型定义，因此更好的做法是编写一个脚本：


```ts
// app.module.ts

import { GraphQLDefinitionsFactory } from '@nestjs/graphql';
import { join } from 'path';

const definitionsFactory = new GraphQLDefinitionsFactory();
definitionsFactory.generate({
  typePaths: ['./src/**/*.graphql'],
  path: join(process.cwd(), 'src/graphql.ts'),
  outputAs: 'class',
});
```

然后我们在 `package.json` 中增加一个 script：


```js
"gen-typings": "ts-node src/generate-typings.ts",
```

现在我们运行 `npm run gen-typings` 之后就会自动识别 `*.graphql` 文件并更新类型定义了：



<p align=center><img src="https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c47cc322a02c409ead9f7f696e855adc~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1664&h=1246&s=551720&e=gif&f=132&b=212524" alt="20240817171523_rec_.gif"/></p>


# Code 优先与 Schema 优先


上面我们演示了如何使用 Schema 优先方式构建了我们的应用，使用 Schema 优先方式允许我们可以使用常规的方式来编写 GraphQL Schema 代码，Nest 允许我们编写多个，最终这些文件会在内存中合并。

Nest 也提供给我们另一种方式，即 Code 优先的方式来构建 GraphQL 应用。在 Code 优先方法中，我们不必编写 GraphQL Schema 文件，而单纯编写 TypeScript 代码，这样避免在 TypeScript 语法和 GraphQL 语法中进行切换。

接下来我们看一下在 Code 优先方式中，我们如何来编写上面的代码，代码参见：https://github.com/L-Qun/state-management-collection/tree/main/examples/graphql/nestjs-code-first

与 Schema 优先一样，首先我们需要配置 `GraphQLModule`，如果我们希望它帮助我们生成 Schema，需要增加 `autoSchemaFile` 对应具体生成的路径：


```ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
}),
```

这样 Nest 就会自动帮助我们生成 Schema 文件：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/51851bf05a914c00a05fe8c05d94c33a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=3044&h=1128&s=376352&e=png&b=292d35)


如果你希望只是加载到内存中，而不生成实际的文件，只需要指定 `autoSchemaFile` 为 `true` 即可：


```ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: true,
}),
```

在 Schema 优先方法中，我们需要定义 Schema 文件，而在 Code 优先方法中我们需要使用 TypeScript 类来定义。例如我们现在有这样一个 Schema：


```graphql
type Book {
  title: String!
  year: Int
}
```

对应在 TypeScript 中等价定义为：


```ts
import { Field, Int, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Book {
  @Field({ description: 'The title of the book.' })
  title: string;

  @Field((type) => Int, { nullable: true })
  year: number;
}
```

让我们来一一解释一下这段代码。

- `@ObjectType()`：装饰器用于将一个 TypeScript 类标记为 GraphQL 对象类型。这个对象类型最终会转换成 GraphQL 模式中的一个类型定义，在这里也就是 `Book` 类型。
- `@Field()` 装饰器用于标记类的属性为 GraphQL 字段，这些字段会被包含在对应的 GraphQL 对象类型中。我们也可以传入 `description` 用来生成文档，比如我们定义了 `@Field({ description: 'The title of the book.' })` 最终可以在 Playground 中看到 `title` 的描述：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2401ae65f9b44e1c89f32e2098239f36~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=3456&h=1984&s=355037&e=png&b=fefefe)

- 类型定义：因为在 TS 中 `number` 类型可能会映射到 GraphQL 中 `Int` 或 `Float` 类型，因此当类型为 `Int` 或 `Float` 时我们需要定义具体类型，而 `string` 和 `boolean` 类型不需要。
- `nullable`：用于指定字段是否可为空，默认是非空的，如果允许非空的，我们需要传入 `nullable: true`。





好，然后我们来将上面案例改写为 Code 优先方式，首先我们编写 `Book` 的类型：


```ts
// /src/books/models/book.model.ts

import { Field, Int, ObjectType } from '@nestjs/graphql';
import { Author } from './author.model';

@ObjectType()
export class Book {
  @Field({ nullable: true })
  title: string;

  @Field((type) => [Author], { nullable: true })
  authors: Author[];

  @Field((type) => Int, { nullable: true })
  year: number;
}
```

对应我们也需要 `Author` 的类型定义：

```ts
// /src/books/models/author.model.ts

import { Field, ObjectType } from '@nestjs/graphql';
import { Book } from './book.model';

@ObjectType()
export class Author {
  @Field()
  name: string;

  @Field((type) => [Book])
  books: Book[];

  @Field()
  nationality: string;
}
```

类型我们定义好了，接下来我们需要完成 Resolver：


```ts
import { Args, Int, Mutation, Query, Resolver } from '@nestjs/graphql';
import { authors, books } from './mock';
import { Book } from './models/book.model';
import { Author } from './models/author.model';

@Resolver()
export class BooksResolver {
  @Query((returns) => [Book])
  books() {
    return books;
  }

  @Query((returns) => [Author], { name: 'authors' })
  getAuthors() {
    return authors;
  }

  @Mutation((returns) => Book)
  addBook(
    @Args('title')
    title: string,
    @Args('year', {
      type: () => Int,
    })
    year: number,
    @Args('authorNames', {
      type: () => [String],
    })
    authorNames: string[],
  ) {
    const newBook = {
      title,
      year,
      authors: authors.filter((author) => authorNames.includes(author.name)),
    };
    books.push(newBook);
    return newBook;
  }
}
```

可以看到在 `@Query()` 装饰器中我们传入了额外的参数，例如 `(returns) => Book`、`(returns) => Author`，这其实对应了原先我们在 Schema 中定义的 `Query`：


```graphql
type Query {
  books: [Book]
  authors: [Author]
}
```

也就是说，为了 Nest 能够正确识别类型，我们需要更显式的传入类型定义。

在上面的例子中当查询 books 数据时，我们会在 `books` 函数全部处理好后返回，但有些时候某些字段不能直接拿到，因此更优雅的一种方式是采用 `@ResolveField()`，比如书的信息中包含了书本的发行时间，而发行时间是基于作者信息额外进行查询的，我们就可以这样：


```ts
@ResolveField()
async year(@Parent() book: Book) {
  const map = {
    book1: 2010,
    book2: 2011,
    book3: 2012,
  };
  // 模拟异步查询过程
  return new Promise((resolve) => setTimeout(resolve, 1000, map[book.title]));
}
```


![20240818184835_rec_.gif](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/90ddd49a6004478d9e3817327e228c70~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1902&h=1090&s=153970&e=gif&f=33&b=142325)

在这里书发行时间是根据书名来异步查询得到的，因此我们需要拿到书的信息，也就是需要 `@Parent()` 装饰器。同时我们必须为 `@Resolver()` 装饰器提供一个值来指示哪个类是父类型（即相应的 `ObjectType` 类名），即对应的我们需要在 `@Resolver()` 中增加类型，完整代码：


```ts
import {
  Args,
  Int,
  Mutation,
  Parent,
  Query,
  ResolveField,
  Resolver,
} from '@nestjs/graphql';
import { authors, books } from './mock';
import { Book } from './models/book.model';
import { Author } from './models/author.model';

@Resolver((of) => Book)
export class BooksResolver {
  @Query((returns) => [Book])
  books() {
    return books;
  }

  @Query((returns) => [Author], { name: 'authors' })
  getAuthors() {
    return authors;
  }

  @ResolveField()
  async year(@Parent() book: Book) {
    const map = {
      book1: 2010,
      book2: 2011,
      book3: 2012,
    };
    return new Promise((resolve) => setTimeout(resolve, 1000, map[book.title]));
  }

  @Mutation((returns) => Book)
  addBook(
    @Args('title')
    title: string,
    @Args('year', {
      type: () => Int,
    })
    year: number,
    @Args('authorNames', {
      type: () => [String],
    })
    authorNames: string[],
  ) {
    const newBook = {
      title,
      year,
      authors: authors.filter((author) => authorNames.includes(author.name)),
    };
    books.push(newBook);
    return newBook;
  }
}
```



# 总结


本章借着一个案例，讲解了 GraphQL 在 Nest 的两种方式 —— Code 优先、Schema 优先，以及在这两种方式下分别如何实现这个案例，同时借着这个案例，我们讲解了相关的知识点。

那在我们的代码中如何选择使用 Code 优先还是 Schema 优先呢？

如果你希望避免在多个语言之间切换，选 Code 优先；如果你希望以传统的方式编写 GraphQL Schema，选择 Schema 优先。

最后我们用一张图来演示一下 Schema 优先和 Code 优先在代码上的等价关系：


<p align=center><img src="https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b9c9fd6dd32492c81c894e8c22445d7~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1384&h=1714&s=314329&e=png&b=ffffff" alt="image.png" width="90%" /></p>



