# Talt: Create TypeScript AST from Template

I'm loving TypeScript AST(Abstract Syntax Tree) analysing/manipulation. We can develop powerful dev-tools using AST.

In this entry, I introduce how to generate TypeScript AST node.

## Motivation

TypeScript allows us to transform .ts sources, as known as "Custom Transformer".

```ts
const customTransformerFactory: ts.TransformerFactory = (ctx) => {
  return (node: ts.Node) => {
    return yourManipulateNodeFn(node);
  }
})

const transformResult = ts.transform(originalSource, [
  customTransformerFactory
]);
```

For example, imagine that we want to generate some import statements in the `yourManipulateNodeFn` function and we want to embed dynamic string(which can be got from other procedure) as the module specifier.

```ts
import { SomeThing } from MODULE_LOCATION_PLACEHOLDER;
```

To achieve this, we should use TypeScript AST factory functions like the following example:

```ts
const moduleLocation = getModuleLocation();

const importStatement = ts.factory.createImportDeclaration(
  undefined,
  undefined,
  ts.factory.createImportClause(
    true,
    undefined,
    ts.factory.createNamedImports([
      ts.factory.createImportSpecifier(
        undefined,
        ts.factory.createIdentifier("SomeThing")
      )
    ])
  ),
  ts.factory.createStringLiteral(moduleLocation)
);
```

Oh, factory, factory and factory! Do you want write over 15 lines to create one statement? Is there more simple way?

## Using templates

Babel provides much useful package to generate AST, [@babel/template](https://babeljs.io/docs/en/babel-template).

```js
import template from "@babel/template";
import generate from "@babel/generator";
import * as t from "@babel/types";

const buildRequire = template(`
  var IMPORT_NAME = require(SOURCE);
`);

const ast = buildRequire({
  IMPORT_NAME: t.identifier("myModule"),
  SOURCE: t.stringLiteral("my-module")
});

console.log(generate(ast).code);
```

The template yields babel-formed AST node.

So I've created an npm package to achieve the same thing as this library with TypeScript AST .

https://github.com/Quramy/talt

Using talt, we can rewrite the above example of import statement like the following:

```ts
import { template } from "talt";

const importStatement = template.statement`
  import type { SomeThing } from MODULE_LOCATION_PLACEHOLDER
`;

const moduleLocation = getModuleLocation();

const ast = importStatement({
  MODULE_LOCATION_PLACEHOLDER: ts.factory.createStringLiteral(moduleLocation)
});
```

Like @babel/template, talt's `template` returns a function and the function replaces identifiers in the template's text to provided AST node object.

Here is another example:

```ts
const expression = talt.expression`
  100 + 100
`();

const statement = talt.statement`
  let x = 200 === EXPRESSION
`({ EXPRESSION: expression });
```

This example generates AST node corresponding to `let x = 200 === 100 * 100` .

If you write TypeScript AST manipulation codes, please try talt and send me FB.
