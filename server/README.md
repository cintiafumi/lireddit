# Lireddit tutorial

## Initial Setup
Install:
```bash
npm init -y

yarn add -D @types/node typescript

yarn add -D ts-node

npx tsconfig.json

yarn add -D nodemon
```

Add some scripts to `package.json`
```json
{
  "scripts": {
    "watch": "tsc -w",
    "dev": "nodemon dist/index.js",
    "start": "node dist/index.js",
    "start2": "ts-node src/index.ts",
    "dev2": "nodemon --exec ts-node src/index.ts"
  },
}
```
---

## PostgreSQL

I added a PostgreSQL database in a docker to run this tut.

---

## Mikro-ORM

Install mikro-orm
```bash
yarn add @mikro-orm/cli @mikro-orm/core @mikro-orm/migrations @mikro-orm/postgresql pg
```

The debug mode will be able only in dev env:
```ts
export const __prod__ = process.env.NODE_ENV === 'production'
```

We initialize the `mikro-orm` in our `index.ts` file
```ts
import { MikroORM } from '@mikro-orm/core'
import { __prod__ } from './constants'
import { Post } from './entities/Post'

const main = async () => {
  const orm = await MikroORM.init({
    entities: [Post],
    dbName: 'lireddit',
    type: 'postgresql',
    debug: !__prod__,
  })

  const post = orm.em.create(Post, {title: 'my first post'})
  await orm.em.persistAndFlush(post)
}

main()
```

Just copied the entity `class Book` from docs [mikro-orm](https://mikro-orm.io/docs/defining-entities) and modified a little bit to our `entities/Post.ts`.
```ts
import { Entity, PrimaryKey, Property } from '@mikro-orm/core'

@Entity()
export class Post {
  @PrimaryKey()
  id!: number;

  @Property({ type: 'date' })
  createdAt = new Date();

  @Property({ type: 'date', onUpdate: () => new Date() })
  updatedAt = new Date();

  @Property({ type: 'text' })
  title!: string;
}
```

To create our Post table, we are going to use mikro cli.

Let's create `mikro-orm.config.ts`
```ts
import { __prod__ } from './constants'
import { Post } from './entities/Post'
import { MikroORM } from '@mikro-orm/core'
import path from 'path'

export default {
  migrations: {
    path: path.join(__dirname, './migrations'),
    pattern: /^[\w-]+\d+\.[tj]s$/,
  },
  entities: [Post],
  dbName: 'lireddit',
  type: 'postgresql',
  debug: !__prod__,
} as Parameters<typeof MikroORM.init>[0];
```

So, the `index.ts` will be
```ts
import { MikroORM } from '@mikro-orm/core'
import { Post } from './entities/Post'
import mikroConfig from './mikro-orm.config'

const main = async () => {
  const orm = await MikroORM.init(mikroConfig)

  const post = orm.em.create(Post, {title: 'my first post'})
  await orm.em.persistAndFlush(post)
}

main()
```

Let's create the migration
```bash
npx mikro-orm migration:create
```

Keep running both `watch` and `dev`
```bash
yarn watch

yarn dev
```

To see our Post, we just show in the console for a while.
```ts
import { MikroORM } from '@mikro-orm/core'
import { Post } from './entities/Post'
import mikroConfig from './mikro-orm.config'

const main = async () => {
  const orm = await MikroORM.init(mikroConfig)
  await orm.getMigrator().up()
  // const post = orm.em.create(Post, {title: 'my first post'})
  // await orm.em.persistAndFlush(post)
  const posts = await orm.em.find(Post, {})
  console.log(posts)
}

main()
```

And we got it
```bash
[
  Post {
    id: 1,
    createdAt: 2020-09-13T21:01:59.000Z,
    updatedAt: 2020-09-13T21:01:59.000Z,
    title: 'my first post'
  },
  Post {
    id: 2,
    createdAt: 2020-09-13T21:02:10.000Z,
    updatedAt: 2020-09-13T21:02:10.000Z,
    title: 'my first post'
  }
]
```

## GraphQL

Install

```bash
yarn add express apollo-server-express graphql type-graphql

yarn add -D @types/express
```

Testing the localhost port
```ts
import { MikroORM } from '@mikro-orm/core'
import mikroConfig from './mikro-orm.config'
import express from 'express'

const main = async () => {
  const orm = await MikroORM.init(mikroConfig)
  await orm.getMigrator().up()

  const app = express()
  app.get('/', (_, res) => {
    res.send('hello')
  })
  app.listen(4000, () => {
    console.log('server started on localhost:4000');
  })
}
```

Check the browser at localhost:4000 to see the message.

Add graphql setup to `index.ts`
```ts
  const apolloServer = new ApolloServer({
    schema: await buildSchema({
      resolvers: [HelloResolver],
      validate: false,
    }),
  })

  apolloServer.applyMiddleware({ app })
```

The `HelloResolver`
```ts
import { Resolver, Query } from 'type-graphql'

@Resolver()
export class HelloResolver {
  @Query(() => String)
  hello() {
    return 'hello world'
  }
}
```

We can check the graphql playground at `localhost:4000/graphql`. The query would be
```graphql
{
  hello
}
```

Let's do the Post CRUD. Add the `PostResolver` class
```ts
import { Resolver, Query, Ctx } from 'type-graphql'
import { Post } from '../entities/Post'
import { MyContext } from '../types'

@Resolver()
export class PostResolver {
  @Query(() => [Post])
  posts() {
    return 'bye'
  }
}
```

Our entity is not a graphql type, but it is possible to convert it to a graphql type adding some decorators to the class
```ts
import { Entity, PrimaryKey, Property } from "@mikro-orm/core";
import { ObjectType, Field, Int } from 'type-graphql'

@ObjectType()
@Entity()
export class Post {
  @Field(() => Int)
  @PrimaryKey()
  id!: number;

  @Field(() => String)
  @Property({ type: 'date' })
  createdAt = new Date();

  @Field(() => String)
  @Property({ type: 'date', onUpdate: () => new Date() })
  updatedAt = new Date();

  @Field()
  @Property({ type: 'text' })
  title!: string;
}
```

Here, you can choose what information is going be exposed or not, by passing the decorator `@Field()`.

We add the context to the `apolloServer`, so all resolvers can access the `orm` context
```ts
  const apolloServer = new ApolloServer({
    schema: await buildSchema({
      resolvers: [HelloResolver, PostResolver],
      validate: false,
    }),
    context: () => ({ em: orm.em })
  })
```

Add `MyContext` to the `types.ts`
```ts
import { EntityManager, IDatabaseDriver, Connection } from "@mikro-orm/core";

export type MyContext = {
  em: EntityManager<any> & EntityManager<IDatabaseDriver<Connection>>
}
```

Now it's possible to access the context `em` inside `PostResolver`
```ts
@Resolver()
export class PostResolver {
  @Query(() => [Post])
  posts(@Ctx() { em }: MyContext): Promise<Post[]> {
    return em.find(Post, {})
  }
}
```

Add `reflect-metadata` and put it in the top of `index.ts`
```bash
yarn add reflect-metadata
```

```ts
import 'reflect-metadata'
```

In the browser, we do the query at playground like this
```graphql
{
  posts {
    id
    createdAt
    updatedAt
    title
  }
}
```

Let's do the query to find one specific Post
```ts
  @Query(() => Post, { nullable: true })
  post(
    @Arg('id') id: number,
    @Ctx() { em }: MyContext
  ): Promise<Post | null> {
    return em.findOne(Post, { id })
  }
```

In the playground
```graphql
{
  post(id: 1) {
    title
  }
}
```

Let's add the `createPost` mutation
```ts
  @Mutation(() => Post)
  async createPost(
    @Arg('title') title: string,
    @Ctx() { em }: MyContext
  ): Promise<Post> {
    const post = em.create(Post, { title })
    await em.persistAndFlush(post)
    return post
  }
```

In the playground we always have to write `mutation` before
```graphql
mutation {
  createPost(title: "post from graphql") {
    id
    createdAt
    updatedAt
    title
  }
}
```

Now, let's add `updatePost` mutation
```ts
  @Mutation(() => Post, { nullable: true })
  async updatePost(
    @Arg('id') id: number,
    @Arg('title', () => String, { nullable: true }) title: string,
    @Ctx() { em }: MyContext
  ): Promise<Post | null> {
    const post = await em.findOne(Post, { id })
    if (!post) {
      return null
    }
    if (typeof title !== 'undefined') {
      post.title = title
      await em.persistAndFlush(post)
    }
    return post
  }
```

And in the playground
```graphql
mutation {
  updatePost(id: 1, title: "waaa") {
    id
    createdAt
    updatedAt
    title
  }
}
```

The `deletePost` mutation
```ts
  @Mutation(() => Boolean)
  async deletePost(
    @Arg('id') id: number,
    @Ctx() { em }: MyContext
  ): Promise<boolean> {
    try {
      await em.nativeDelete(Post, { id })
    } catch {
      return false
    }
    return true
  }
```

`deletePost` playground
```graphql
mutation {
  deletePost(id: 2)
}
```
