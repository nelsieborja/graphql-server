# Getting Started

## Creating the project

### Open a terminal and run the following commands:

```shell
$ mkdir hackernews-node
  cd hackernews-node
  yarn init -y
```

This will create a folder called `hackernews-node` and initializes with a `package.json` file inside the folder.

> `package.json` is a configuration file for an Application. It lists all the dependencies and other configuration options such as _scripts_, needed by the App

## Creating a raw GraphQL server

1. ### Create the entry point for the GraphQL server:

   ```shell
   $ mkdir src
     touch src/index.js
   ```

   The entry file will be named as `index.js` and lives inside the folder `src`.

2. ### Add GraphQL server dependency to the project:

   ```shell
   $ yarn add graphql-yoga
   ```

3. ### Add the following code to the entry file:

   ```js
   // -------------------
   // src/index.js
   // -------------------
   const { GraphQLServer } = require('graphql-yoga');
   // 1
   const typeDefs = `
     type Query {
       info: String!
     }
   `;
   // 2
   const resolvers = {
     Query: {
       info: () => `This is the API of a Hackernews Clone`
     }
   };
   // 3
   const server = new GraphQLServer({
     typeDefs,
     resolvers
   });
   server.start(() =>
     console.log(`Server is running on http://localhost:4000`)
   );
   ```

   - `[1]` - `typeDefs` constant defines the GraphQL schema

     > Every GraphQL schema has 3 special _root types_ - `Query`, `Mutation`, and `Subscription`. The field on these root types are called _root field_ and define the available API operations (eg `info` from the above sample)

   - `[2]` - `resolvers` object defines the actual implementation of the GraphQL schema. Its structure is identical to the structure of the type definitions inside `typeDefs`

     > The underlying `graphql-js` reference implementation ensures that the return types of the `resolvers` adhere to the type definitions from the GraphQL schema

   - `[3]` - schema and resolvers are bundled and passed to the `GraphQLServer` which is imported from `graphql-yoga`. This tells the server what API operations are accepted and how they should be resolved

## Testing the GraphQL server

### Run the following command in the root directory of the App:

```shell
$ node src/index.js
```

As indicated by the terminal output, the server is now running on `http://localhost:4000`. What you will see in the browser is the [GraphQL Playground](https://github.com/graphcool/graphql-playground) ("GraphQL IDE") which consists:

- **Left pane**: Where you write your _queries_, _mutations_ and _subscription_
- **Right pane**: Server response will be displayed here
- **Play button** (or `CMD+ENTER`): It will send the query written on the left pane to the server
- **SCHEMA button**: This will reveal an auto-generated documentation based on the schema definition, API operations & data types of the App
