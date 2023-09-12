# p2

Hi everyone.

My name is Quramy.
And, I am a web application developer loving GraphQL.

It's query about me.

# p3

Today, I talk about how to explain contextual precondition of GraphQL schema.

# p4

I think that a GraphQL schema works as a contract between server and client.

For server-side, schema should implement to resolve query result for valid operations.
And for client-side, client applications should send operations defined by schema.

# p5

By the way, Have your heard "Design by Contract" ?

DbC is a programming paradigm.
And this is pasted from Wikipedia...

DbC has these principals, Preconditions and Postconditions.

Whenever client application guarantees preconditions, sever provides postconditions.

# p6

This figure maps GraphQL components to DbC terms.

GraphQL Operations, fields and variables, correspond to "Preconditions".

Executable schema, in other words resolver implementation, correspond to "Software Components",

And resolved JSON data correspond to "Postconditions".

# p7

GraphQL specification guarantees the followings:

Field names in operation are defined in schema.
The values of the filed variables are valid types.

If the client violates these preconditions, GraphQL engine throws validation exception.

# p8

Next, let's think about server implementations.

We know that GraphQL field resolver takes four arguments:
Object resolved by parent, field arguments, context, and metadata.

So, this is today's problem,,,

How to give precondition for the 3rd argument, context?

# p9

This. (pointing)

# p10

Often, we want to assume preconditions for context.
For example,

First, mutation field executable only for authenticated user attached specific authorization.
Next, another example, some query fields executable only for staging environment

And, if these preconditions are established, the postconditions get more sharp and offensive.

# p11

Here are two schema examples, the first is defensive and the second is offensive.

The offensive schema can provide strict string because the context is guaranteed to execute by the precondition.

# p12

Next problem is that“How to expose contextual precondition to schema?".

My answer is "using schema directives", like this (pointing).

# p13

If you use graphql ruby, you can define directives like this (pointing).

# p14

And we can use AST Analyzer class to check the directive's precondition like this (pointing).

The `on_enter_field` method is called back for each field in the operation, and executes the logic corresponding to the directive.

# p15

The directive separates the assertion logic from the resolver implementation, which is separation of concern.
In other words, we can recognize the directive assertion logic as an aspect.

So, we can annotate this directive if we add more fields which needs the same precondition, because aspect is reusable.

# p16

There is a caveat to use schema directive.

For now, GraphQL schema directives can not be introspected.
So, custom schema directives are only "structured documents" for clients.

If you want more details see this GraphQL spec issue (pointing)

# p17

Here is today conclusion.

GraphQL schema is "contract" between server and client.
Preconditions make application components more sharp and offensive.
We can define schema directives to explain precondition for execution context.
Custom scheme directives can be recognized as an Aspect .

# p18

Thank you!
