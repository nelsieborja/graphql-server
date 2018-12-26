# Realtime GraphQL Subscriptions

> `Subscription` is a GraphQL feature that allows a server to send data to its client when a specific _event_ happens

## Subscriptions with `Prisma`

Prisma comes with out-of-the-box support for subscriptions. For each model in the _Prisma datamodel_ you can subscribe to the following _events_:

- A new model is **created**
- An existing model **updated**
- An existing model is **deleted**

You can subscribe to these events using the `$subscribe` method of the _Prisma client_.

> `$subscribe` property proxies the subscriptions from the _Prisma API_. It is used to resolve subscriptions and push the event data

## Subscribing to new `Link` elements

1. ### Extend the GraphQL schema

   Just like _queries_ and _mutations_, the first step to implement a subscription is to extend the GraphQL schema definition. Add the following `Subscription` type definition in `schema.graphql` file:

   ```js
   // -------------------
   // src/schema.graphql
   // -------------------
   type Subscription {
     newLink: Link
   }
   ```

2. ### Add the resolver for subscriptions

   The resolvers for subscriptions are slightly different from _queries_ and _mutations_:

   - They return an `AsyncIterator` rather than a data
   - They are wrapped inside an object and need to be provided as the value for a `subscribe` key in the object. You also need to provide another key called `resolve` that returns the data from the data emitted by the `AsyncIterator`

   Create a new file called `Subcription.js` and add the following implementation in the file:

   ```js
   // -------------------
   // src/resolvers/Subcription.js
   // -------------------
   function newLinkSubscribe(parent, args, context, info) {
     return context.prisma.$subscribe.link({ mutation_in: ['CREATED'] }).node();
   }

   const newLink = {
     subscribe: newLinkSubscribe,
     resolve: payload => {
       return payload;
     }
   };

   module.exports = {
     newLink
   };
   ```

3. ### Include `Subscription` resolver in the main `resolvers` object

   Import the module at the top of `index.js` file:

   ```js
   // -------------------
   // src/index.js
   // -------------------
   const Subscription = require('./resolvers/Subscription');
   ```

   Still on the same file, update the definition of the `resolvers` object to look as follows:

   ```js
   const resolvers = {
     Query,
     Mutation,
     Subscription,
     User,
     Link
   };
   ```

## Testing subscriptions

Again, restart the GraphQL server...

Open a new Playground and send the following subscription:

```js
subscription {
  newLink {
    id
    url
    description
    postedBy {
      id
      name
      email
    }
  }
}
```

You'll not immediately see the result of the operation, instead you get a loading spinner indicating that it's waiting for an event to happen. Also a text "Listening..." is displayed below the pane.

Now send the following `post` mutation in another Playground to trigger a subscription event. Make sure the request is authenticated:

```js
mutation {
  post(
    url: "www.graphqlweekly.com"
    description: "Curated GraphQL content coming to your email inbox every Friday"
  ) {
    id
  }
}
```

You should see a result in the Playground where the subscription is running.

## Adding a voting feature

### Implementing a `vote` mutation

1. Extend the _Prisma datamodel_ to represent votes in the database. To do so, adjust `datamodel.prisma` file to look as follows:

   ```js
   // -------------------
   // prisma/datamodel.prisma
   // -------------------
   type Link {
     // ...
     votes: [Vote!]!
   }

   type User {
     // ...
     votes: [Vote!]!
   }

   type Vote {
     id: ID! @unique
     link: Link!
     user: User!
   }
   ```

   `Vote` type has one-to-many relationships to the `User` and `Link` types.

2. To apply the changes and update the _Prisma client API_ to add CRUD operations for `Vote` type, don't forget to deploy the service again with:

   ```shell
   $ prisma deploy
   ```

3. Extend the GraphQL schema definition to expose a `vote` mutation in the GraphQL server:

   ```js
   // -------------------
   // src/schema.graphql
   // -------------------
   type Mutation {
     // ...
     vote(linkId: ID!): Vote
   }
   ```

   The referenced `Vote` type also needs to be defined in there:

   ```js
   type Vote {
    id: ID!
    link: Link!
    user: User!
   }
   ```

   It should also be possible to query all `votes` from a `Link`, so adjust the `Link` type:

   ```js
   type Link {
     // ...
     votes: [Vote!]!
   }
   ```

### Implementing `Vote` resolver

1. Add the following resolver implementation to `Mutation.js` file:

   ```js
   // -------------------
   // src/resolvers/Mutation.js
   // -------------------
   async function vote(parent, { linkId }, context, info) {
     // 1
     const userId = getUserId(context);
     // 2
     const linkExists = await context.prisma.$exists.vote({
       user: { id: userId },
       link: { id: linkId }
     });
     if (linkExists) {
       throw new Error(`Already voted for link: ${linkId}`);
     }
     // 3
     return context.prisma.createVote({
       user: { connect: { id: userId } },
       link: { connect: { id: linkId } }
     });

     module.exports = {
       // ...
       // 4
       vote
     };
   }
   ```

   - `1` - similar to `post` resolver, it validates the incoming JWT with `getUserId()`. If it's valid the function returns the `userId` of the `User`, otherwise it throws an error
   - `2` - `$exists` function is generated per model by _Prisma client_ instance. It takes a `where` filter object where you can specify a condition that applies to an element of that type. In this case, is to verify if the requesting `User` (`user:{id:userId}`) has not yet voted the `Link` (`link:{id:args.linkId}`)
   - `3` - `createVote` is used to create a new `Vote` that's _connected_ to the `User` and the `Link`
   - `4` - exports the resolver

2. Account for the new relations in the GraphQL schema:

   - `votes` on `Link`

     Add the following function to `Link.js` file:

     ```js
     // -------------------
     // src/resolvers/Link.js
     // -------------------
     function votes(parent, args, context) {
       return context.prisma.link({ id: parent.id }).votes();
     }

     module.exports = {
       // ...
       votes
     };
     ```

   - `user` and `link` on `Vote`

     Create a new file called `Vote.js` inside `resolvers` folder, and add the following code to it:

     ```js
     // -------------------
     // src/resolvers/Vote.js
     // -------------------
     function link(parent, args, context) {
       return context.prisma.vote({ id: parent.id }).link();
     }

     function user(parent, args, context) {
       return context.prisma.vote({ id: parent.id }).user();
     }

     module.exports = {
       link,
       user
     };
     ```

3. Finally, include `Vote` resolver in the main `resolvers` object in `index.js` file:

   ```js
   // -------------------
   // src/index.js
   // -------------------
   const Vote = require('./resolvers/Vote');

   const resolvers = {
     // ...
     Vote
   };
   ```

### Subscribing to new votes

The approach is analogous to the `newLink` query.

1. Add a new field called `newVote` to the `Subscription` type of the GraphQL schema:

   ```js
   // -------------------
   // src/schema.graphql
   // -------------------
   type Subscription {
     newLink: Link
     newVote: Vote
   }
   ```

2. Add its resolver in `Subscription.js` file:

   ```js
   // -------------------
   // src/resolvers/Subscription.js
   // -------------------
   // ...

   function newVoteSubscribe(parent, args, context, info) {
     return context.prisma.$subscribe.vote({ mutation_in: ['CREATED'] }).node();
   }
   const newVote = {
     subscribe: newVoteSubscribe,
     resolve: payload => payload
   };

   module.exports = {
     // ...
     newVote
   };
   ```

### Testing the `newVote` subscription

Use the following subscription:

```js
subscription {
  newVote {
    id
    link {
      url
      description
    }
    user {
      name
      email
    }
  }
}
```

Test it with the following `vote` mutation. Make sure to replace `__LINK_ID__` placeholder with the `id` of an actual `Link` from the database and that the request is authenticated:

```js
mutation {
  vote(linkId: "__LINK_ID__") {
    link {
      url
      description
    }
    user {
      name
      email
    }
  }
}
```
