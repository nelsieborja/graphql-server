# Adding a Database - `Prisma`

`Prisma` provides a convenient data access layer which is taking care of resolving queries for you. When using Prisma, resolvers would be like forwarding the incoming queries to the underlying Prisma engine which in turn resolves the query against the actual database.

## Architecture

- `Prisma server` - provides the _data access layer_ to the App; makes the _API server_ communication to database easy via _Prisma_
- `API server` - built previously with `graphql-yoga`
- `Prisma client` - consumes the API of the _Prisma server_ inside the _API server_

## Setting up Prisma with a demo database

1. ### Create the required folder/files with the following commands:

   ```shell
   $ mkdir prisma
   touch prisma/prisma.yml
   touch prisma/datamodel.prisma
   ```

   This creates a new folder called `prisma` and adds the following files into the folder:

   - `prisma.yml` - main/root configuration file for the _Prisma server_
   - `datamodel.prisma` - contains the definition of the [datamodel](https://www.prisma.io/docs/1.23/datamodel-and-migrations/datamodel-MYSQL-knul/)

     > `datamodel` defines App's _models_ where each model is mapped to a table in the underlying database. It is the foundation for the auto-generated CRUD and realtime operations of the Prisma API

2. ### Create the first model - `Link`

   Prisma also uses [GraphQL SDL](https://www.prisma.io/blog/graphql-sdl-schema-definition-language-6755bcb9ce51) for model definitions, so the existing `Link` definition from `src/schema.graphql` can just be copied into `prisma/datamodel.prisma`:

   ```js
   // -------------------
   // prisma/datamodel.prism
   // -------------------
   type Link {
     id: ID! @unique
     createdAt: DateTime!
     description: String!
     url: String!
   }
   ```

   The difference of it from the previous `Link` would be:

   - `@unique` - a directive that tells Prisma the value of such field should never get duplicated
     - `id: ID!` - special field in the _Prisma datamodel_; its value is auto-generated
   - `createdAt: DateTime!` - stores the time when a `Link` is created; also manage by Prisma and is read-only in the API

3. ### Create the _template_ of the Prisma server

   Add the following contents to `prisma/prisma.yml`:

   ```yml
   endpoint: ''
   datamodel: datamodel.prisma
   generate:
     - generator: javascript-client
       output: ../src/generated/prisma-client
   ```

   - `endpoint` - HTTP endpoint for the _Prisma API_
   - `datamodel` - points to the datamodel file
   - `generate` - specifies which language the _Prisma client_ should be generated; `output` specifies where it will be located

4. ### Install the Prisma CLI

   Before you can even deploy the server, need to install the Prisma CLI globally with the following command:

   ```shell
   $ yarn global add prisma
   ```

5. ### Time to deploy the server!

   Run the following command in the root directory of the App:

   ```shell
   $ prisma deploy
   ```

   This will use a free _demo database_ ([AWS Aurora](https://aws.amazon.com/de/rds/aurora/)) hosted in Prisma Cloud. Learn [here](https://www.prisma.io/docs/-a002/) more about setting up Prisma locally or with a database.

   The `prisma deploy` command starts an interactive process:

   - Select the **Demo server**. When the browser opens, **register with Prisma Cloud** and go back to terminal
   - Select **Region** for the demo server. Hit enter twice to use the suggested values for **service** and **stage**

   > [Prisma is open-source](https://github.com/prisma/prisma). You can deploy it with [Docker](http://docker.com/) to a cloud provider of your choice (Digital Ocean, AWS, Google Cloud, etc)

   Once completed, the CLI writes the endpoint for the _Prisma API_ to `prisma.yml` file.

6. ### Lastly, generate the _Prisma client_ for the datamodel

   > _Prisma client_ is an auto-generated client library that lets you read/write data in the database through the _Prisma API_

   Run the following command (inside `prisma` folder) to generate it, this will read the information from `prisma.yml` and gerenates the _Prisma client_ accordingly:

   ```shell
   $ prisma generate
   ```

   This also generates a file with TypeScript definitions (`index.d.ts`), it will help you with auto-completion when reading/writing data using the _Prisma client_.

---

The _Prisma API_ is only an interface to the database, but doesn't allow for any sort of application logic (ie: data validation, authentication, 3rd party services) which is needed in most Apps.
