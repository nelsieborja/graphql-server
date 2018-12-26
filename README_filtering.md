# Filtering, Pagination & Sorting

## Filtering via `where`

1. ### Add the `filter` string to the `feed` query in the GraphQL schema:

   ```js
   // -------------------
   // src/schema.graphql
   // -------------------
   type Query {
     info: String!
     feed(filter: String): [Link!]!
   }
   ```

2. ### Update the `feed` resolver to account for the new parameter

   ```js
   // -------------------
   // src/resolvers/Query.js
   // -------------------
   async function feed(parent, { filter }, context, info) {
     const where = filter
       ? {
           OR: [{ description_contains: filter }, { url_contains: filter }]
         }
       : {};

     const links = await context.prisma.links({ where });
     return links;
   }
   ```

   If no `filter` string is provided, the `where` will be just an empty object thus no filter out is going to happen on the `Link` elements. In case it has, the `where` object is constructed with two filter conditions.

3. ### Test the filter functionality with the following query:

   ```js
   feed(filter: "graphql") {
     id
     description
     url
     postedBy {
       id
       name
     }
   }
   ```

## Pagination via `first`, `skip` and `last`

On a high-level, there are two major approaches on how it can be tackled:

- **Limit-Offset**: Request a specific _chunk_ by providing the start index _(offset)_ and a count of items to be retrieved _(limit)_.
- **Cursor-based**: Clients provide the cursor (every element in the list is associated with a unique ID or the cursor) of the starting element and a count of items to be retrieved.

Prisma supports both these approaches (read more from their [docs](https://www.prisma.io/docs/-rsc2#pagination) or learn more about the approaches [here](https://blog.apollographql.com/understanding-pagination-rest-graphql-and-relay-b10f835549e7)). Let's implement **limit-offset** pagination:

- `first` - the _limit_, means it will grab the _first_ x elements after a provided start index
- `skip` - the _start index_; default is `0`
- `last` - the _last_ x elements

1. ### Adjust the `feed` query to accept `skip` and `first` args:

   ```js
   // -------------------
   // src/schema.graphql
   // -------------------
   type Query {
     info: String!
     feed(filter: String, skip: Int, first: Int): [Link!]!
   }
   ```

2. ### Adjust the implementation of the `feed` resolver:

   ```js
   // -------------------
   // src/resolvers/Query.js
   // -------------------
   async function feed(parent, { skip, first }, context, info) {
     // ...
     const links = await context.prisma.links({
       where,
       skip,
       first
     });
     return links;
   }
   ```

3. ### Test the pagination API with the following query:

   ```js
   feed(
     first: 1
     skip: 1
   ) {
     id
     description
     url
   }
   ```

   This returns the second `link` from the list.

## Sorting

Include the ordering options from the Prisma API in the API of the GraphQL server. To do so, create an `enum` that represents all the ordering options:

1. ### Add the `enum` definition to the GraphQL schema:

   ```js
   // -------------------
   // src/schema.graphql
   // -------------------
   enum LinkOrderByInput {
     description_ASC
     description_DESC
     url_ASC
     url_DESC
     createdAt_ASC
     createdAt_DESC
   }
   ```

   It represents the various ways how the list of `Link` elements can be sorted.

2. ### Adjust the `feed` query to include the `orderBy` argument:

   ```js
   // -------------------
   // src/schema.graphql
   // -------------------
   type Query {
     info: String!
     feed(
       filter: String
       skip: Int
       first: Int
       orderBy: LinkOrderByInput
     ): [Link!]!
   }
   ```

3. ### Update the `feed` resolver to pass the `orderBy` argument along to Prisma

   The implementation is similar to the pagination API:

   ```js
   // -------------------
   // src/resolvers/Query.js
   // -------------------
   async function feed(parent, { skip, first, orderBy }, context, info) {
     // ...
     const links = await context.prisma.links({
       where,
       skip,
       first,
       orderBy
     });
     return links;
   }
   ```

4. ### Sample query that sorts the returned `links` by their creation dates:
   ```js
   query {
     feed(orderBy: createdAt_DESC) {
       id
       description
       url
     }
   }
   ```

## Returning the total count of `Link` elements

1. ### Create a new type called `Feed` and point `feed` field to it:

   ```js
   // -------------------
   // src/schema.graphql
   // -------------------
   type Query {
     info: String!
     feed(filter: String, skip: Int, first: Int, orderBy: LinkOrderByInput): Feed!
   }

   type Feed {
     links: [Link!]!
     count: Int!
   }
   ```

2. ### Adjust the `feed` resolver again:

   ```js
   // -------------------
   // src/resolvers/Query.js
   // -------------------
   async function feed(parent, args, context, info) {
     // ...
     const count = await context.prisma
       .linksConnection({
         where
       })
       .aggregate()
       .count();

     return {
       links,
       count
     };
   }
   ```

   - Uses `linksConnection` query from _Prisma client API_ to retrieve the total number of `Link` elements stored in the database
   - `links` and `count` are then wrapped in an object to adhere to the `Feed` type that's just added to the GraphQL schema

3. ### Test the revamped `feed` query as follows:

   ```js
   query {
     feed {
       count
       links {
         id
         description
         url
       }
     }
   }
   ```
