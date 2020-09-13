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

I added a PostgresQL database in a docker to run this tut.

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

Install micro-orm
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
import { Entity, PrimaryKey, Property } from "@mikro-orm/core";

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

Let's create `micro-orm.config.ts`
```ts
import { __prod__ } from './constants';
import { Post } from './entities/Post';
import { MikroORM } from '@mikro-orm/core';
import path from 'path';

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
import microConfig from './mikro-orm.config'

const main = async () => {
  const orm = await MikroORM.init(microConfig)

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
import microConfig from './mikro-orm.config'

const main = async () => {
  const orm = await MikroORM.init(microConfig)
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
