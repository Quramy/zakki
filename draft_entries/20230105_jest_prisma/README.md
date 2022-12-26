# Test your Prisma app Part 1: Isolated transaction via jest-prisma

I develop and maintain Node.js services using Prisma ORM in my work.

First of all, I'm loving Prisma because it has strongly type safety query APIs and various useful generators.
On the other hand, I'm feeling Prisma lacks some developer-experience about testing.

So I build and publish 2 npm packages about Prisma testing. In this post, I introduce a part of them.

- https://github.com/Quramy/jest-prisma

## Issue in integrated testing Prisma app

It's not hard to test Prisma app using "real" database.
[Prisma doc](https://www.prisma.io/docs/guides/testing/integration-testing) also says how to run Jest's test suite communicating with PostgreSQL via docker-compose.
And it's not hard to run them in your CI environment too.
For example, if you want to test your Prisma app in GitHub Actions, you can use [service containers](https://docs.github.com/en/actions/using-containerized-services/about-service-containers#example-mapping-redis-ports).

However, Docker Compose or GHA's service containers don't resolve the following issues:

- You should initialize DB after each test case runs. In other words, you should truncate all tables and run seeds in `beforeEach` .
- You can not run test suites in parallel because a test suite may change DB and affect another test suite running. So you should run with Jest with `--maxWorkers=1` option.

These problem is not because of Node.js nor Prisma. They're general problems caused by using real database.

So, let's see how other languages / frameworks tackle them.

Below is a quote from [Spring test](https://docs.spring.io/spring-framework/docs/current/reference/html/testing.html#testing-tx), an application framework for Java.

> One common issue in tests that access a real database is their effect on the state of the persistence store.
>
> The TestContext framework addresses this issue. By default, the framework creates and rolls back a transaction for each test. You can write code that can assume the existence of a transaction.

Yes, **the framework creates and rolls back a transaction for each test**.

And the same feature exists in other language. RSpec Rails in Ruby provides an option, [`use_transactional_fixture`](https://relishapp.com/rspec/rspec-rails/docs/transactions).

## jest-prisma: Transaction managed by framework

I want feature like these frameworks to use in Jest and Prisma. So I build an npm package:

https://github.com/Quramy/jest-prisma

jest-prisma is a test environment of Jest.

```js
/* jest.config.mjs */
export default {
  // ... Your jest configuration

  testEnvironment: "@quramy/jest-prisma/environment"
  // You can choose jest-prisma-node too.
  // testEnvironment: "@quramy/jest-prisma-node/environment"
};
```

You can access Prisma client instance managed by jest-prisma via `jestPrisma.cilent`.

The following example has 2 test cases and they are run in sequence. See the 2nd case, "Count user".
The case asserts `prisma.user.count()` to be zero and succeeds because that jest-prisma rolled back the 1st case which inserts a record to user table.

```ts
describe(UserService, () => {
  // jestPrisma.client works with transaction rolled-back automatically after each test case end.
  const prisma = jestPrisma.client;

  test("Add user", async () => {
    const createdUser = await prisma.user.create({
      data: {
        id: "001",
        name: "quramy"
      }
    });

    expect(
      await prisma.user.findFirst({
        where: {
          name: "quramy"
        }
      })
    ).toStrictEqual(createdUser);
  });

  test("Count user", async () => {
    expect(await prisma.user.count()).toBe(0);
  });
});
```

## Under the hood

jest-prisma implements isolated transactional fixture using [Prisma's interactive transaction](https://www.prisma.io/docs/concepts/components/prisma-client/transactions#interactive-transactions).

```ts
await prisma.$transaction(async clientInTransaction => {
  // Statements in the callback is executed 1 transaction.

  // If callback is rejected, the transaction is rolled back.
  throw new Error();
});
```

jest-prisma calls the `$transaction` and sets the `clientInTransaction` to `jestPrisma.client` before each test case starts.

Interactive transaction was introduced as a preview feature in Prisma client v2.29 and generally available since Prisma client v4.7.0.
So if your `@prisma/client` version is l.e. 4.6, add the following your `schema.primsa` file to use jest-prisma.

```graphql
generator client {
  provider = "prisma-client-js"
  previewFeatures = ["interactiveTransaction"]
}
```

## Summary

I made an npm package, jest-prisma.

- It creates transaction which is rolled back automatically after each test case run.
- So, each test case does not affect each other. You can run your test suites in parallel.

Please check it out and send to me feedback.

I'll introduce another Primsa utility package in the next post.
