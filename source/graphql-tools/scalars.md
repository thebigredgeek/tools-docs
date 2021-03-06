---
title: Custom scalars
order: 309
description: Add custom scalars to your graphql-tools generated schema.
---

The GraphQL specification includes the following default scalar types: `String`, `Int`, `Float` and `Boolean`. While this covers most of the user cases, often you need to support custom atomic data types (e.g. Date), or you want a version of an existing type that does some validation. To enable this, GraphQL allows you to define custom scalar types.

To define a custom scalar you simply add it to the schema string with the following notation:

```js
scalar MyCustomScalar
```

Afterwards, you need to define the behavior of the scalar by supplying three functions:

1. `__serialize`: How the value should be serialized as JSON when sent to the client in the query result.
2. `__parseValue`: How the value should be parsed from JSON when received from the client in the form of query variables.
3. `__parseLiteral`: How the value should be parsed from the query AST when received from the client as an inline argument in the query.

These are the same as the methods you would use with GraphQL.js directly, but in `graphql-tools` they are prefixed with a double underscore to differentiate from field names.

Note that [Apollo Client does not currently have a way to automatically interpret custom scalars](https://github.com/apollostack/apollo-client/issues/585), so there's no way to automatically reverse the serialization on the client.

## Examples

Let's look at a couple of examples to demonstrate the potential of custom scalars.

### Date as a scalar

The goal is to define a `Date` data type for returning `Date` values from the database. Let's say we're using a MongoDB driver that uses the native JavaScript `Date` data type. The `Date` data type can be easily serialized as a number using the [`getTime()` method](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/getTime). Therefore, we would like our GraphQL server to send and receive `Date`s as numbers when serializing to JSON. This number will be resolved to a `Date` on the server representing the date value. On the client, the user can simply create a new date from the received numeric value.

The following is the implementation of the `Date` data type. First, the schema:

```js
scalar Date

type MyType {
   created: Date
}
```

Next, the resolver:

```js
import { Kind } from 'graphql/language';

const resolverMap = {
  Date: {
    __parseValue(value) {
      return new Date(value); // value from the client
    },
    __serialize(value) {
      return value.getTime(); // value sent to the client
    },
    __parseLiteral(ast) {
      if (ast.kind === Kind.INT) {
        return parseInt(ast.value, 10); // ast value is always in string format
      }
      return null;
    },
  },
};
```

### Validations

In this example, we follow the [official GraphQL documentation](http://graphql.org/docs/api-reference-type-system/) for the scalar datatype. Let's say that you have a database field that should only contain odd numbers. First, the schema:

```js
scalar Odd

type MyType {
    oddValue: Odd
}
```

Next, the resolver:

```js
import { Kind } from 'graphql/language';

Odd: {
  __serialize: oddValue,
  __parseValue: oddValue,
  __parseLiteral(ast) {
    if (ast.kind === Kind.INT) {
      return oddValue(parseInt(ast.value, 10));
    }
    return null;
  }
}

function oddValue(value) {
  return value % 2 === 1 ? value : null;
}
```

### JSON as a scalar

While we usually want to define a schema for our data, in some cases it makes sense to store unstructured objects in the database and not define a GraphQL schema for it. JSON is a commonly used format for storing such objects. In GraphQL, we can define a custom scalar type to serialize and parse JSON:


```js
scalar JSON

type MyType {
   jsonField: JSON
}
```

And the implementation of the resolver:

```js
import { Kind } from 'graphql/language';

function parseJSONLiteral(ast) {
  switch (ast.kind) {
    case Kind.STRING:
    case Kind.BOOLEAN:
      return ast.value;
    case Kind.INT:
    case Kind.FLOAT:
      return parseFloat(ast.value);
    case Kind.OBJECT: {
      const value = Object.create(null);
      ast.fields.forEach(field => {
        value[field.name.value] = parseJSONLiteral(field.value);
      });

      return value;
    }
    case Kind.LIST:
      return ast.values.map(parseJSONLiteral);
    default:
      return null;
  }
}

const resolvers =
  JSON: {
    __parseLiteral: parseJSONLiteral,
    __serialize: value => value,
    __parseValue: value => value,
  },
};
```

For more information please refer to the [official documentation](http://graphql.org/graphql-js/type/) or to the [Learning GraphQL](https://github.com/mugli/learning-graphql/blob/master/7.%20Deep%20Dive%20into%20GraphQL%20Type%20System.md) tutorial.
