# Connecting Server and Database via Prisma client

## Updating the resolver functions to use [Prisma client](https://www.prisma.io/docs/prisma-client/)

1. ### Start with cleanup!

   Remove the `links` array and `idCount` variable, these data will now be stored in an actual database.

2. ### Update the implementation of resolver functions

   Remove the references for deleted variables and return actual data from the database. The `resolvers` object should now look as follows:

   ```js
   // -------------------
   // src/index.js
   // -------------------
   const resolvers = {
     Query: {
       feed: (root, args, context, info) => {
         return context.prisma.links();
       }
     },

     Mutation: {
       post: (root, { url, description }, context, info) => {
         return context.prisma.createLink({
           url,
           description
         });
       }
     }
   };
   ```

   - ### The `context` argument

     A plain JS object, that a resolver in the resolver chain can read from and write to - thus a means for resolvers to communicate.

   - ### Understanding the `feed` resolver

     It accesses a `prisma` object on `context` - this `prisma` is a _Prisma client_ instance that's imported from the generated `prisma-client` library.

     > _Prisma client_ exposes CRUD API for the models in the datamodel, for reading/writing in the database. They are auto-generated based on the definitions written in `datamodel.prisma` file

   - ### Understanding the `post` resolver

     Similar to `feed` resolver, it's invoking a function on the _Prisma client_ instance.

## Attaching the generated Prisma client to `context`

1. ### Install the dependency to the App

   This dependency is required to make the auto-generated _Prisma client_ to worked. So to install, run the following command in the root directory of the App:

   ```shell
   $ yarn add prisma-client-lib
   ```

2. ### Attach the generated _Prisma client_ to the `context`

   By Attaching it to the `context`, the resolvers will finally get access to it. First, import the instance into `src/index.js` by adding the following import statement at the top of the file:

   ```js
   // -------------------
   // src/index.js
   // -------------------
   const { prisma } = require('./generated/prisma-client');
   ```


    Then attach it to the `context` when `GraphQLServer` is being initialized. To do so, update the `GraphQLServer` instantiation to look as follows:

    ```js
    const server = new GraphQLServer({
      typeDefs: './src/schema.graphql',
      resolvers,
      context: { prisma }
    });
    ```

    The `context` object (3rd argument in resolver functions) is actually being initialized here. Since _Prisma client_ instance is now attached to it, `context.prisma` becomes accessible in the resolvers.

## Testing the new implementation

As usual, need to restart the GraphQL server. Once it's started, same `feed` query and `post` mutation as before can be sent to the server. The difference is that this time the submitted links are <ins>persisted in the Prisma Cloud demo database</ins>.

---

The _Prisma client instance_ translates the GraphQL operations from the Prisma API into JS functions.
