# Adding Queries

## schema-driven

`schema-driven` (or `schema-first development`) is a process that should be followed when adding a new feature to the API:

1. ### Extending the schema definition

   Extend the GraphQL schema definition with a new _root field_ (and new _data types_, if needed):

   ```js
   // -------------------
   // src/index.js
   // -------------------
   const typeDefs = `
   // 1
   type Query {
     info: String!
     // 2
     feed: [Link!]!
   }
   // 3
   type Link {
     id: ID!
     description: String!
     url: String!
   }
   `;
   ```

   - `[1]` - all _queries_ should go inside `type Query{}`
   - `[2]` - `feed` _root field_ is added to the _Query type_; will be used for retrieving the list of `Link` elements
   - `[3]` - `Link` _data type_ represents the links that can be posted to the App
   - `!` - indicates the field will never contain any elements that are `null`

2. ### Implement resolver functions

   The next step is to implement the resolver function for the `feed` query, still on the same file:

   ```js
   // 1
   let links = [
     {
       id: 'link-0',
       url: 'graphql.org'
       description: 'GraphQL is awesome!',
     }
   ];
   const resolvers = {
     // 2
     Query: {
       info: () => 'This is the API of a Hackernews Clone',
       // 3
       feed: () => links
     },
     // 4
     Link: {
       id: root => root.id,
       description: root => root.description,
       url: root => root.url
     }
   };
   ```

   - `[1]` - `links` variable stores the links at runtime. For now, it stored only _in-memory_ rather than persisted in a database
   - `[2]` - all `resolvers` for _Query_ should go inside `Query:{}`
   - `[3]` - resolver for the `feed` root field, named after its corresponding field from the schema definition
   - `[4]` - resolvers for the fields on the `Link` type. This can actually be removed, they are not needed because GraphQL server infers what they look like

3) ### Testing the query

   Restart the server with `CMD+C` (to kill the server) and by running `node src/index.js` to start it again.

   Send the following `feed` query through the Playground:

   ```js
   query {
     feed {
       id
       url
       description
     }
     info
   }
   ```

   Notice that you're also sending `info` query, as you can actually send multiple queries in one request. A sample server response would be:

   ```js
   {
     "data": {
       "feed": [
         {
           "id": "link-0",
           "url": "graphql.org",
           "description": "GraphQL is awesome!"
         }
       ],
       "info": "This is the API of a Hackernews Clone"
     }
   }
   ```

## The query resolution process

Every field inside schema definition is backed by one _resolver function_ which is responsible for returning the precise data for that field. Server invokes all these _resolver functions_ for the fields that are contained in the query, package up the response according to the query's shape.

> ---
>
> ### All resolver functions _always_ receive 4 arguments:
>
> - `root` (or `parent`) - the result of the previous resolver function
> - `args` - carries the _`arguments`_ for the GraphQL operation
> - `context` - _explained in the next section_
> - `info` - _learn about it from [here](https://blog.graph.cool/graphql-server-basics-the-schema-ac5e2950214e) and [here](https://blog.graph.cool/graphql-server-basics-demystifying-the-info-argument-in-graphql-resolvers-6f26249f613a)_
>
> ---

GraphQL queries can be nested, each level of nesting (ie: nested curly braces) corresponds to one _resolver execution level_. Consider the following query:

```js
query {
  feed {
    id
    description
    url
  }
}
```

The first level invokes the `feed` resolver and returns the data stored in `links`. On the second level, the resolver for `Link` type for each element from the returned list gets invoked, hence the incoming `root` object on this level is the element inside the `links` list.
