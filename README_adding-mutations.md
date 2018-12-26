# Adding Mutations

## Extending the schema definition

Like adding _`queries`_, need to start by adding the new operations to the GraphQL schema definition:

```js
// -------------------
// src/index.js
// -------------------
const typeDefs = `
  type Query {
    info: String!
    feed: [Link!]!
  }
  // 1
  type Mutation {
    // 2
    post(url: String!, description: String!): Link!
  }
  type Link {
    id: ID!
    description: String!
    url: String!
  }
`;
```

- `[1]` - all _mutations_ should go inside `type Mutation{}`
- `[2]` - `post` field will be used for adding a new `Link` element; it expects for 2 _arguments_ (`url` and `description`) and returns a `Link` element

### Refactor the App

1. Copy the schema definition defined in `typeDefs` into a new file - `src/schema.graphql.js`:

   ```js
   // -------------------
   // src/schema.graphql.js
   // -------------------
   type Query {
     info: String!
     feed: [Link!]!
   }

   type Mutation {
     post(url: String!, description: String!): Link!
   }

   type Link {
     id: ID!
     description: String!
     url: String!
   }
   ```

2. Delete `typeDefs` after updating the `GraphQLServer` instantiation as follows:
   ```js
   // -------------------
   // src/index.js
   // -------------------
   const server = new GraphQLServer({
     typeDefs: './src/schema.graphql',
     resolvers
   });
   ```

One useful option of the `GraphQLServer` constructor is that `typeDefs` can be provided either <ins>directly as a _**string**_</ins> or <ins>by _**referencing a file**_</ins> that contains the schema definition

## Implementing the resolver function

Next step is to implement the _resolver function_ for the new field `post`:

```js
// -------------------
// src/index.js
// -------------------

//...

// 1
let idCount = links.length;
const resolvers = {
  // ...

  // 2
  Mutation: {
    // 3
    post: (root, { url, description }) => {
      // 4
      const link = {
        id: isCount + 1,
        url,
        description
      };
      // 5
      links.push(link);
      // 6
      return link;
    }
  }
};
```

- `[1]` - `idCount` variable serves as a way to generate unique IDs for newly created `Link` elements
- `[2]` - all `resolvers` for _Mutation_ should go inside `Mutation:{}`
- `[3]` - `post` function implements the `resolver` for the `post` field
- `[4]` - creates a new `link` object
- `[5]` - adds it to the existing `links` list
- `[6]` - returns the new `link`

## Testing the mutation

Restart the server so that the new API operations would reflect. Send the following mutation through the Playground:

```js
// Client query
mutation {
  post(
    url: "www.prisma.io"
    description: "Prisma turns your database into a GraphQL API"
  ) {
    id
  }
}

// Server response
{
  "data": {
    "post": {
      "id": "link-1"
    }
  }
}
```

To verify the mutation worked, send the `feed` query from before:

```js
// Client query
query {
  feed {
    id
    url
    description
  }
}

// Server response
{
  "data": {
    "feed": [
      {
        "id": "link-0",
        "url": "graphql.org",
        "description": "GraphQL is awesome!"
      },
      {
        "id": "link-1",
        "url": "www.prisma.io",
        "description": "Prisma turns your database into a GraphQL API"
      }
  }
}
```
