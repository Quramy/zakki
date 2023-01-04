# Test your Prisma app Part 2: prisma-fabbrica, test data factory utility

In the previous post, I introduced how to test Prisma app with isolated transaction via jest-prisma.
In this post, I introduce another package, prisma-fabbrica, which helps to create test data.

https://github.com/Quramy/prisma-fabbrica

## Motivation

We need to set up test data sufficient for each test case. Especially with integrated testing, in other words using a real database, we should insert test data as precondition.

For example, imagine the following Prisma schema, which represents that a `User` model has many posts and a `Post` model belongs to a user.

```graphql
model User {
  id    String @id
  email String @unique
  name  String
  posts Post[]
}

model Post {
  id       String @id
  title    String
  author   User   @relation(fields: [authorId], references: [id])
  authorId String
}
```

And if you want to test some logic to update title of a post such as:

```ts
/* src/updatePostTitle.ts */

export async function updatePostTitle(postId: string, title: string) {
  return await prisma.post.update({
    where: {
      id: postId
    },
    data: {
      title
    }
  });
}
```

Test code for the above function is the following:

```ts
/* src/updatePostTitle.test.ts */

import { updatePostTitle } from "./updatePostTitle";

test("updatePostTitle", async () => {
  await prisma.post.create({
    data: {
      id: "post",
      title: "title",
      author: {
        create: {
          data: {
            id: "user001",
            name: "Quramy",
            email: "Quramy@myservice.com"
          }
        }
      }
    }
  });

  await expect(updatePostTitle("post", "new title")).resolve.toMatchObject({
    title: "new title"
  });
});
```

Okay, the above test works and passes. But take careful at the code. It contains some useless information.
The `post.create` block contains a user model fields although the test case is not interested in post's author.

```ts
await prisma.post.create({
  data: {
    id: "post",
    title: "title",
    author: {
      create: {
        data: {
          id: "user001",
          name: "Quramy",
          email: "Quramy@myservice.com"
        }
      }
    }
  }
});
```

The above code lines increases as the number of required fields in `User` model increases.

I don't like to set up unneeded models (e.g. `author` in the above case) and I want to **keep tests to contain necessary and sufficient information**.

## Test data factory

I built prisma-fabbrica inspired from factory_bot, a utility library to set up Active Record models in Ruby.

The above test code for `updatePostTitle` can be rewritten as the following using prisma-fabbrica:

```ts
/* src/updatePostTitle.test.ts */

import { updatePostTitle } from "./updatePostTitle";
import { PostFactory } from "./factories";

test("updatePostTitle", async () => {
  const post = await PostFactory.create({ title: "title" });

  await expect(updatePostTitle(post.id, "new title")).resolve.toMatchObject({
    title: "new title"
  });
});
```

```ts
/* src/factories.ts */

import { defineUserFactory, definePostFactory } from "./__generated__/fabbrica";

export const UserFactory = defineUserFactory();

export const PostFactory = definePostFactory({
  defaultData: {
    author: UserFactory
  }
});
```

prisma-fabbrica is a Prisma generator and generates functions to define model factory(e.g. `defineUserFactory` and `definePostFactory`).

Model factories work as "shared fixture". So you can re-use model factories in multiple test cases once you define them.

## Features of prisma-fabbrica

### Auto complete required scalar fields

The `User` model has some required scalar fields, `id`, `email` and `name`

```graphql
model User {
  id    String @id
  email String @unique
  name  String
  posts Post[]
}
```

But you don't need give the required values. Factory automatically fills values corresponding their types.

```ts
await UserFactory.create();
// or
await UserFactory.create({ email: "test@example.com" });
// or
await UserFactory.create({ id: "user001", email: "test@example.com" });
```

### Relations

You can order to create relations in various ways.

```ts
const PostFactory = definePostFactory({
  defaultData: { author: UserFactory }
});

await PostFactory.create(); // Create not only post but also user
```

```ts
// Create a user that has 2 posts
await UserFactory({
  posts: {
    create: await PostFactory.buildList(2)
  }
});
```

```ts
const PostFactory = definePostFactory(async () => ({
  defaultData: {
    author: connectOrCreate({
      where: { id: "user" },
      create: await UserFactory.build({
        id: "user"
      })
    })
  }
}));

// The following posts belongs to the same user
await PostFactory.create();
await PostFactory.create();
```

See [document](https://github.com/Quramy/prisma-fabbrica#usage-of-factories) if you want more details.

## Summary

In this post, I introduced how to write Primsa integrated test codes efficiently via prisma-fabbrica.

Finally, I put an example repository using jest-prisma, explained in my previous post, and prisma-fabbrica.

https://github.com/Quramy/prisma-yoga-example/blob/main/tests/resolvers/query.test.ts

I hope this helps you.
