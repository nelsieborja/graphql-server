# Authentication

## Adding a `User` model

1. ### Add `User` type to the _Prisma datamodel_

   Need to also add the _relation_ between the `User` and the already existing `Link` type to express that `Link`s are _posted_ by `User`s. Update `datamode.prisma` file to look as follows:

   ```js
   // -------------------
   // prisma/datamode.prisma
   // -------------------
   type Link {
      id: ID! @unique
      createdAt: DateTime!
      description: String!
      url: String!
      // 1
      postedBy: User
    }
    // 2
    type User {
      id: ID! @unique
      name: String!
      email: String! @unique
      password: String!
      // 3
      links: [Link!]!
    }
   ```

   - `[1]` - `postedBy` is a _relation field_ which points to a `User` instance
   - `[2]` - `User` model
   - `[3]` - a list of `Link`s; expresses a one-to-many realationship using SDL

2. ### Redeploy the _Prisma API_

   This is to update the _Prisma API_ with the new change made on the _datamodel_ and to migrate the underlying database schema. Run the same command as before:

   ```shell
   $ prisma deploy
   ```

3. ### Need to also update the _Prisma client_

   This will expose CRUD methods for the newly added `User` model. Run the same command as before within `prisma` folder:

   ```shell
   $ prisma generate
   ```

4. ### Run `prisma generate` automatically

   To get rid of the **3rd step**, configure a [post-deployment hook](https://www.prisma.io/docs/prisma-cli-and-configuration/prisma-yml-5cy7/#hooks-optional) that gets invoked every time after `prisma deploy` is ran. To do so, add the hook at the end of `prisma/prisma.yml` file:

   ```yml
   hooks:
     post-deploy:
       - prisma generate
   ```

## Extending the GraphQL schema

1. ### With `schema-driven` process in mind, add `signup` and `login` mutations

   Update the `Mutation` type in `schema.graphql` file as follows:

   ```js
   // -------------------
   // src/schema.graphql.js
   // -------------------
   type Mutation {
     post(url: String!, description: String!): Link!
     signup(email: String!, password: String!, name: String!): AuthPayload
     login(email: String!, password: String!): AuthPayload
   }
   ```

2. ### Add required types - `AuthPayload` and `User` types

   Add the `AuthPayload` along with a `User` type definition to the schema:

   ```js
   // -------------------
   // src/schema.graphql.js
   // -------------------
   type AuthPayload {
     token: String
     user: User
   }

   type User {
     id: ID!
     name: String!
     email: String!
     links: [Link!]!
   }
   ```

   `signup` and `login` mutations behave similarly - both return information about the `User` as well as a `token`, this information is bundled in `AuthPayload` type.

3. ### Express _bi-directional_ relation between `User` and `Link`

   Achieve this by adding a `postedBy` field (that points to a `User` instance) to existing `Link` type definition:

   ```js
   // -------------------
   // src/schema.graphql.js
   // -------------------
   type Link {
     id: ID!
     description: String!
     url: String!
     postedBy: User
   }
   ```

## Implementing the resolver functions

Before doing that, let's refactor the code to keep it more modular - pull out the resolvers for each type into their own files.

1. ### Pull out the resolvers into a new file

   Run the following command in the terminal to create a folder called `resolvers` and add new files (`Query.js`, `Mutation.js`, `User.js` and `Link.js`) to it:

   ```shell
   $ mkdir src/resolvers
   touch src/resolvers/Query.js
   touch src/resolvers/Mutation.js
   touch src/resolvers/User.js
   touch src/resolvers/Link.js
   ```

2. ### Move the implementation of `feed` resolver into `Query.js`:

   ```js
   // -------------------
   // src/resolvers/Query.js
   // -------------------
   function feed(parent, args, context, info) {
     return context.prisma.links();
   }

   module.exports = {
     feed
   };
   ```

3. ### Add the resolvers to `Mutation.js`:

   ```js
   // -------------------
   // src/resolvers/Mutation.js
   // -------------------
   async function signup(parent, args, context, info) {
     // 1
     const password = await brcyt.hash(args.password, 10);
     // 2
     const user = await context.prisma.createUser({ ...args, password });
     // 3
     const token = jwt.sign({ userId: user.id }, APP_SECRET);
     // 4
     return {
       token,
       user
     };
   }
   ```

   - `[1]` - encyrpts the `User`'s password using the `bcryptjs` library (will install later)
   - `[2]` - uses the _Prisma client instance_ to store the new `User` in the database
   - `[3]` - generates a JWT which is signed with an `APP_SECRET` (will create `APP_SECRET` and install `jwt` library later)
   - `[4]` - returns the `token` and `user` in an object that adheres to the shape of an `AuthPayload` object from the GraphQL schema

   ```js
   // -------------------
   // src/resolvers/Mutation.js
   // -------------------
   async function login(parent, { email, password }, context, info) {
     // 1
     const user = context.prisma.user({ email });
     if (!user) {
       throw new Error('No such user found');
     }
     // 2
     const valid = await bcrypt.compare(password, user.password);
     if (!valid) {
       throw new Error('Invalid password');
     }
     // 3
     const token = jwt.sign({ userId: user.id }, APP_SECRET);
     // 4
     return {
       token,
       user
     };
   }
   ```

   - `[1]` - uses the _Prisma client instance_ to retrieve an existing `User` by passing the `email` argument in the `login` mutation. If no `user` with that `email` was found, it throws an error
   - `[2]` - compares the provided `password` with the one that's stored in the database. If they don't match, it throws an error
   - `[3]` & `[4]` - do the same as `signup`'s

   ```js
   module.exports = {
     signup,
     login
   };
   ```

   - finally, this will export the `signup` and `login` modules

4. ### Add the required dependencies to the App

   Install the tools (`brcyt` & `jwt`) used before with the following command:

   ```shell
   $ yarn add jsonwebtoken bcryptjs
   ```

5. ### Create reusable utilities

   Pull out the codes that are being reused in few places into a new file called `utils.js`, added inside the folder `src`:

   ```js
   // -------------------
   // src/utils.js
   // -------------------
   const jwt = require('jsonwebtoken');

   const APP_SECRET = 'GraphQL-is-aw3some';

   function getUserId(context) {
     // 1
     const Authorization = context.request.get('Authorization');
     if (Authorization) {
       // 2
       const token = Authorization.replace('Bearer ', '');
       // 3
       const { userId } = jwt.verify(token, APP_SECRET);
       return userId;
     }
     // 4
     throw new Error('Not authenticated');
   }

   module.exports = {
     APP_SECRET,
     getUserId
   };
   ```

   - `APP_SECRET` - used to sign the JWTs issued to users
   - `getUserId` - helper function which will be called in resolvers which require authentication
   - `[1]` - retrieves the `Authorization` header (which contains the `User`'s JWT) from the context. Note that `request` is yet to be attached to the `context`
   - `[2]` - extracts the token
   - `[3]` - verifies the JWT and retrieves `User`'s ID from it
   - `[4]` - throws error if process is not successful, thereby protects the resolver

6. ### Utilize the utilities

   Complete the implementation of _Mutation resolvers_ by adding the following import statements to the top of `Mutation.js` file:

   ```js
   // -------------------
   // src/resolvers/Mutation.js
   // -------------------
   const bcrypt = require('bcryptjs');
   const jwt = require('jsonwebtoken');
   const { APP_SECRET, getUserId } = require('../utils');
   ```

7. ### Attach `request` to the `context`

   One more thing to do is to attach the `request` object to the `context`. To do so, adjust the instantiation of the `GraphQLServer` in `index.js` file as follows:

   ```js
   // -------------------
   // src/index.js
   // -------------------
   const server = new GraphQLServer({
     typeDefs: './src/schema.graphql',
     resolvers,
     context: request => {
       return {
         ...request,
         prisma
       };
     }
   });
   ```

   The _return value_ of the `context` is changed from object to function (which returns the `context`). This approach attaches the HTTP request that carries the incoming GraphQL operations to the `context` - makes the `Authorization` header accessible from resolvers.

## Requiring authentication for the `post` mutation

### Add the following resolver implementation for `post` in `Mutation.js` file:

```js
// -------------------
// src/resolvers/Mutation.js
// -------------------
function post(parent, { url, description }, context, info) {
  const userId = getUserId(context);
  return context.prisma.createLink({
    url,
    description,
    postedBy: { connect: { id: userId } }
  });
}
```

- `getUserId` - retrieves the ID of the `User`. This ID is stored in the JWT that's set at the `Authorization` header of the incoming request, hence you know which `User` is creating the `Link`. In case of unsuccessful retrieval of `userId`, the function scope is exited before the `createLink` mutation is invoked - the GraphQL response would be an authentication error.
- `connect` - connects the `Link` to be created with the `User` who is creating it

### Resolving relations in GraphQL API

1. To resolve `postedBy` relation on `Link`, add the following code to `Link.js` file:

   ```js
   // -------------------
   // src/resolvers/Link.js
   // -------------------
   function postedBy(parent, args, context) {
     return context.prisma.link({ id: parent.id }).postedBy();
   }

   module.exports = {
     postedBy
   };
   ```

   The resolver is fetching `Link` using the _Prisma client_ instance and then invoked `postedBy` on it - this resolves the `postedBy` field from the `Link` type in GraphQL schema.

2. The `links` relation on `User` can be resolved in a similar way, add the following code to `User.js` file:

   ```js
   // -------------------
   // src/resolvers/User.js
   // -------------------
   function links(parent, args, context) {
     return context.prisma.user({ id: parent.id }).links();
   }

   module.exports = {
     links
   };
   ```

### Putting it together

1. Use the new resolver implementations in `index.js` file. To do so, copy the following import statements to the top of the file:

   ```js
   // -------------------
   // src/index.js
   // -------------------
   const Query = require('./resolvers/Query');
   const Mutation = require('./resolvers/Mutation');
   const User = require('./resolvers/User');
   const Link = require('./resolvers/Link');
   ```

2. Update the definition of the `resolvers` object to look as follows:
   ```js
   const resolvers = {
     Query,
     Mutation,
     User,
     Link
   };
   ```

## Testing the authentication flow

Restart the GraphQL server, as usual...

The very first thing to do is to test the `signup` mutation, to create a new `User` in the database:

```js
mutation {
  sugnup(
    name: "Nelsie"
    email: "nelsieborja@gmail.com"
    password: "graphql"
  ) {
    token
    user {
      id
    }
  }
}
```

From the server response, copy the authentication `token` and open the **HTTP HEADER** pane. Specify the `Authorization` header by pasting the following snippet there, with the **TOKEN** placeholder replaced with the copied token:

```js
{
  "Authorization": "Bearer __TOKEN__"
}
```

With the `Authorization` header in place, send the following to the GraphQL server:

```js
mutation {
  post(
    url: "www.graphqlconf.org"
    descriptions: "An awesome GraphQL conference"
  )
}
```

This will invoke the `post` resolver, therefore validates the provided JWT. Also the new created `Link` is now connected to the `User` that was sent before with the `signup` mutation.

To verify everything worked, send the following `login` mutation:

```js
mutation {
  login(
    email: "nelsieborja@gmail.com"
    password: "graphql"
  ) {
    token
    user {
      email
      links {
        url
        description
      }
    }
  }
}
```

A sample server response would be:

```js
{
  "data": {
    "login": {
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9",
      "user": {
        "email": "nelsieborja@gmail.com",
        "links": [
          {
            url: "www.graphqlconf.org",
            descriptions: "An awesome GraphQL conference"
          }
        ]
      }
    }
  }
}
```

---

To hide potentially sensitive information from client applications, you can redefine a `type` in the application schema instead of importing from the Prisma database schema.
