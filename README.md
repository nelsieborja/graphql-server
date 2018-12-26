# GraphQL Server with Node.js

## Technologies

- `graphql-yoga` - GraphQL server library built on top of _Express_, `apollo-server`, `graphql-js` and more
- [Prisma](https://www.prisma.io/) - Replaces traditional ORMs. Use the Prisma client to implement your GraphQL resolvers and simplify database access
- [GraphQL Playground](https://github.com/graphcool/graphql-playground) - "GraphQL IDE" that allows interactive exploration on GraphQL API. Somewhat similar to `Postman` _(for REST APIs)_
- `prisma-client-lib` - Required to run Prisma client in JS
- [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken)
- [bcryptjs](https://github.com/dcodeIO/bcrypt.js)

---

## Steps

I have rewritten my own version in order to get a better grasp of how things work and with the aim to summarize and rephrase each step in a way that _"I"_ can easily and quickly understand it.

- [Getting Started](README_getting-started.md)
- [Adding Queries](README_adding-queries.md)
- [Adding Mutations](README_adding-mutations.md)
- [Adding a Database](README_adding-database.md)
- [Connecting Server & Database via Prisma client](README_prisma-client.md)
- [Authentication](README_authentication.md)
- [Realtime GraphQL Subscriptions](README_subscriptions.md)
- [Filtering, Pagination & Sorting](README_filtering.md)

---

## Reference

[HOW TO GRAPHQL](https://www.howtographql.com/graphql-js/1-getting-started/)
