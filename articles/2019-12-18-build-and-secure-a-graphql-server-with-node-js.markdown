---
layout: post
title: Build and Secure a GraphQL Server with Node.js
metatitle: Build and Secure a GraphQL Server with Node.js
description: 'Learn how to handle authentication and authorization of a GraphQL server using Node.js and JWTs.'
metadescription: 'How to handle authentication and authorization with GraphQL is often neglected. Learn how to build and secure a GraphQL server with Node.js using JWTs.'
date: 2019-12-18 08:30
category: Developers, Tutorial, GraphQL
post_length: 4
community_topic_id: 35314
author:
  name: Roy Derks
  url: https://twitter.com/gethackteam
  avatar: https://cdn.auth0.com/blog/guest-authors/roy-derks.png
design:
  illustration: https://cdn.auth0.com/blog/illustrations/graphql.png
tags:
  - nodejs
  - graphql
  - mongodb
  - authentication
  - authorization
  - server-side
  - api
related:
  - 2019-08-13-vuejs-spring-boot-kotlin-and-graphql-building-modern-apps-part-1
  - 2019-08-14-authorization-series-pt-3-dynamic-authorization-with-graphql-and-rules
  - 2019-03-13-developing-and-securing-graphql-apis-with-laravel
---

**TL;DR:** Since its public release in 2015, GraphQL has grown from a new technology into a mature API specification, which is used by both small and big tech companies worldwide. Using a type system, it lets you query and mutate data using a simple and understandable format.

Although many articles online demonstrate how to create a GraphQL server with Node.js, how to handle authentication and authorization is often neglected. This post will show how to build a GraphQL server and consequently secure it using Auth0.

## Prerequisites

Before you continue reading this tutorial, you'll need to make sure that you've Node.js and npm installed on your machine. If you don't have these installed yet, you can find the installation instructions [here](https://nodejs.org/en/download/).

You will, of course, need to have some prior knowledge of JavaScript. Also, you will need an Auth0 authorization server to implement real-world authentication and authorization. As such, I invite you to <a href="https://auth0.com/signup" data-amp-replace="CLIENT_ID" data-amp-addparams="anonId=CLIENT_ID(cid-scope-cookie-fallback-name)">sign up for a free Auth0 account here</a>.

## What is GraphQL?

In this tutorial, you'll build a GraphQL server with Node.js that uses Auth0 to handle authentication and authorization. But to build this server, you need to learn more about what GraphQL is and how it works. If you're already familiar with GraphQL and its principles, you can immediately proceed to the next section of this tutorial.

GraphQL can best be described as a query language for APIs that works with the principle _"Ask for what you need, get exactly that."_ With GraphQL, you can send so-called documents containing operations (either queries, mutations, or subscriptions) to a GraphQL server, and the response of the server will follow the same structure of those documents. These three types of operations all use the GraphQL query language to return the same predictable format. Also, all operations are sent to the same endpoint no matter what data you're trying to query or mutate.

An example of such a document containing a `query` operation would be:

```graphql
query {
  person(id: 12) {
    name
    age
  }
}
```

This query is for retrieving a person with the `id` 12 from a GraphQL server and will return a JSON output with the same structure. For the example above, the following (mock) data will be returned:

```json
{
  "data": {
    "person": {
      "name": "John Doe",
      "age": 43
    }
  }
}
```

Different than with REST APIs, this means the users are in control of the data response as they determine its format with their GraphQL document and not the API itself. How to structure those documents is found in the schema of the GraphQL API that receives and handles them. A schema is structured with types and defines what the returned data look like and which operations can be sent &mdash; as can be seen in the example schema for the previous document:

```graphql
type Person {
  id: ID
  name: String
  age: Int
}
type Query {
  person(id: Int): Person
}
```

According to this schema, you can send a query to retrieve a person with the identifier `id` and ask for every field of the type `Person`. The output of this query will, as shown before, follow the structure of the query and, therefore, the structure of the type `Person`. The data for the fields `name` and `age` will consequently always be returned with the types as defined in the schema above. The hierarchical distinction between object types like `Person` and built-in scalar types like `String` is that the first ones can be used to contain other fields.

As mentioned before, your documents are sent to just one (often called) `/graphql` endpoint for every type of operation in your schema. Even if you define relationships in your document, something that you'll do later on in this tutorial, the same endpoint is used to retrieve this data. This is also a clear advantage over REST APIs, as they often require you to send multiple requests to different endpoints on the server to gather relational data.

## Building a GraphQL Server with Node.js

In this section, you'll build a basic GraphQL server with Node.js, that uses data from a mock MongoDB instance. How to secure that server isn't handled until the next section, where you'll use JWTs and Auth0 to authenticate and authorize users and the requests they send.

### Creating a GraphQL server with Express

With Express, you can easily create a fresh Node.js server with limited code, that is performant enough to serve as a GraphQL server. Before you can create the Express server, you'll need to create a new project on your machine. Start by creating a new directory from your terminal by running:

```
mkdir auth0-graphql-server
```

Afterward, move into this directory and initiate a new project using npm with the following commands:

```
cd auth0-graphql-server
npm init -y
```

This last command will create the new project in the directory `auth0-graphql-server` and a fresh `package.json` file.

> You can also execute the command `npm init` without the `-y` flag, which requires you to answer several questions to set up the project, such as name, author, etc.

Now that you’ve created the initial project, the packages for building the GraphQL server can be installed:

```
npm install express express-graphql graphql
```

These packages are [`express`](https://www.npmjs.com/package/express) and [`graphql`](https://www.npmjs.com/package/graphql) itself, and the [`express-graphql`](https://www.npmjs.com/package/express-graphql) middleware that makes it possible to use your Express server with GraphQL. Also, all the dependencies of these packages are installed, as you can see from the output in your terminal.

The actual code that runs the GraphQL server must be added in a new file in this project, which you can create in a new directory called `src`. To create this directory and this file, you can run the commands below or use your preferred text editor or IDE.

```
mkdir src
touch src/index.js
```

And subsequently, copy and paste the following code into this file:

```js
// src/index.js

const express = require('express');
const graphqlHTTP = require('express-graphql');
const { buildSchema } = require('graphql');

// Construct a schema, using GraphQL schema language
const schema = buildSchema(`
  type Query {
    hello: String
  }
`);

// Provide resolver functions for your schema fields
const resolvers = {
  hello: () => 'Hello world!',
};

const app = express();
app.use(
  '/graphql',
  graphqlHTTP({
    schema,
    rootValue: resolvers,
  }),
);
app.listen(4000);

console.log(`🚀 Server ready at http://localhost:4000/graphql`);
```

In this file, the packages to create the server and to use GraphQL are imported, the schema and resolvers are created, and the GraphQL server is exposed to port `4000`. If you look at the schema, one query can be sent to the GraphQL server that's returning a field with the type `String`. The data that's returned by that query can be found in the resolvers, which are functions that are responsible for the data of the operations. Resolvers are _"where the magic happens"_ in a GraphQL server, as they contain the logic to retrieve data from a data source like a database. Also, resolvers are often pure functions and should return the data in the same format as defined in the schema.

You can test this by using `node` to execute this file:

```
node src/index.js
```

That should return the following message in your terminal, meaning you're able to send documents to the GraphQL server:

```
🚀 Server ready at http://localhost:4000/graphql
```

This is, however, not the most practical solution to run your GraphQL server, as every change to the code would require a restart of the server. Instead of using `node`, you can use `nodemon`, which looks for changes in your code and actively restarts the server. This package can be installed using npm:

```
npm install --save-dev nodemon
```

After the installation has completed, you can add a start script to the `package.json` file to make it even easier for you to start the server:

```json
// package.json
{
  "name": "auth0-graphql-server",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "start": "nodemon src/index.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.17.1",
    "express-graphql": "^0.9.0",
    "graphql": "^14.5.8"
  },
  "devDependencies": {
    "nodemon": "^1.19.4"
  }
}
```

Before running this start script, you need to stop the server from your terminal and then you can just run `npm start` to start the server, which should show the same message in the terminal as before. To send a document containing this query to the GraphQL server, you can send it to the server using a cUrl request from a separate terminal tab or window:

```
curl -X POST -H "Content-Type: application/json" \
--data '{ "query": "{ hello }" }' \
http://localhost:4000/graphql
```

You've now sent your very first GraphQL query to the server, which returned a _"Hello world!"_ string. In the next part of this section, you'll add a more and advanced schema to this server.

### Construct a GraphQL schema

With the basics for the GraphQL server in order, you can now set up the actual schema that you want to use for the server. There are many approaches to define the structure of your schema, one of them being **schema-first**. With the schema-first approach, your schema is defined manually in the GraphQL server first, before the resolvers are connected to this schema.

> As mentioned, there are multiple approaches to define the structure of your schema, and all have advantages and disadvantages. For this tutorial, the schema-first approach provides the most benefits, as the data for the schema is completely mocked, and the project is created from scratch. For a deeper understanding of these approaches, have a look at the article [_The Problems of "Schema-First" GraphQL Server Development_](https://www.prisma.io/blog/the-problems-of-schema-first-graphql-development-x1mn4cb0tyl3) by [Nikolas Burk](https://twitter.com/nikolasburk) from [Prisma](https://twitter.com/prisma).

The GraphQL server you'll create in this article will show and mutate a list of events and its attendees. In the introduction of this tutorial, the type `Person` was already shown, which will also be used in the schema for this GraphQL server. Before extending the schema, let's move its definition to a separate file that you can create in the `src` directory. This file can be called `schema.js` and exports the schema for the server to use. Also, you need to import the `buildSchema` method from `graphql` to create the schema in this file.

To create this new file, you need to run the following from the terminal:

```bash
touch src/schema.js
```

Afterward, copy the following code into the `schema.js` file:

```js
// src/schema.js

const { buildSchema } = require('graphql');

// Construct a schema, using GraphQL schema language
const schema = buildSchema(`
  type Query {
    hello: String
  }
`);

module.exports = schema;
```

In the file `src/index.js`, the schema must now be imported from the file `src/schema.js`, which means that you can delete the import of `buildSchema` and add the declaration of the schema instead:

```js
// src/index.js

const express = require('express');
const graphqlHTTP = require('express-graphql');
const schema = require('./schema');

// Provide resolver functions for your schema fields
const resolvers = {

...
```

As the GraphQL server should return information about events and the people attending these events, the schema can be altered like this in the file `src/schema.js`:

```js
// src/schema.js

const { buildSchema } = require('graphql');

// Construct a schema, using GraphQL schema language
const schema = buildSchema(`
  type Event {
    id: ID!
    title: String!
    description: String
    date: String
    attendants: [Person!]
  }

  type Person {
    id: ID!
    name: String!
    age: Int
  }

  type Query {
    events: [Event!]!
    event(id: Int!): Event!
  }
`);

module.exports = schema;
```

The schema now consists of two queries that will either return an array of events or a specific event based on the argument `id`. The fields that have a type ending with an exclamation mark, like `title: String!` are non-nullable. That means that `title`, in this case, will always return some string, while `attendants: [Person!]` will always return an array that's null or filled with objects of the type `Person`.

When you try and send a document with either of the two queries to the GraphQL server, you'll receive an error, as the resolvers aren't returning data in the structure of the schema yet. This will be added in the next part of this section, where a mock MongoDB instance is used to fill the server with data.

### Connect the resolvers

Your resolvers are the functions that will gather all the data that you request in your documents, but they are indifferent to the data source. For this tutorial, you'll use a mock MongoDB instance to generate the data for your resolvers and thereby make it possible to send the operations from your schema to the server. If you're not familiar with MongoDB, then what you need to know is that it's a NoSQL database based on documents. These documents are formatted with JSON and, therefore, easy to read and also easy to query with JavaScript.

Creating a mock MongoDB instance can be done with the package [`mongodb-memory-server`](https://github.com/nodkz/mongodb-memory-server) that will create a mock database in your application memory. By running the following command from your terminal, you can install both the package for running the mock server and the default MongoDB client:

```
npm install mongodb-memory-server mongodb
```

After adding these packages you need to create a new file in your project that will set up the mock database, you can call this file `database.js` and place it in the directory `src`:

```bash
touch src/database.js
```

In this file, the code block below must be placed, which has all the code you need to create the MongoDB instance:

```js
// src/database.js
const { MongoMemoryServer } = require('mongodb-memory-server');
const { MongoClient } = require('mongodb');

let database = null;

async function startDatabase() {
  const mongo = new MongoMemoryServer();
  const mongoDBURL = await mongo.getConnectionString();
  const connection = await MongoClient.connect(mongoDBURL, {
    useNewUrlParser: true,
  });

  if (!database) {
    database = connection.db();
    await database.collection('events').insertMany([
      {
        id: 1,
        title: 'GraphQL Introduction Night',
        description: 'Introductionary night to GraphQL',
        date: '2019-11-06T17:34:25+00:00',
        attendants: [
          {
            id: 1,
            name: 'Peter',
            age: 34,
          },
          {
            id: 2,
            name: 'Kassandra',
            age: 23,
          },
        ],
      },
      {
        id: 2,
        title: 'GraphQL Introduction Night #2',
        description: 'Introductionary night to GraphQL',
        date: '2019-11-06T17:34:25+00:00',
        attendants: [
          {
            id: 3,
            name: 'Kim',
            age: null,
          },
        ],
      },
    ]);
  }

  return database;
}
module.exports = startDatabase;
```

As you can see in the block of code above, you use the function `startDatabase` to connect with the mock MongoDB instance, populate the instance with mock data for your collections, and return to the function caller a reference to the instance.

This `startDatabase` function is called right after creating the server but also every time that a resolver gets called. Therefore you must check if the database variable already exists, which means that a mock MongoDB instance has already been created. Otherwise, every document with a query or mutation you pass to the server will create a new mock MongoDB instance, and any changes to the data won't be visible.

Connecting your GraphQL server with a data source is often done by using the context function that is available from the resolvers. The context holds important contextual data like database access or user information and is created by the GraphQL server during setup.

In your project, you can add the context and connect with the database by making these changes to the file `src/index.js`:

```js
// src/index.js

const express = require('express');
const graphqlHTTP = require('express-graphql');
const schema = require('./schema');
const startDatabase = require('./database');

// Create a context for holding contextual data
const context = async () => {
  const db = await startDatabase();

  return { db };
};

...

const app = express();
app.use(
  '/graphql',
  graphqlHTTP({
    schema,
    rootValue: resolvers,
    context,
  }),
);
app.listen(4000);

console.log(`🚀 Server ready at http://localhost:4000/graphql`);
```

You import the `database` module, create a `context` to hold contextual data, and add that `context` to the object argument of the `graphqlHTTP` middleware function.

With these additions, the resolvers are now able to access the context function so they can thereby retrieve data from the mock database. To make this possible, you need to use the object `db` from the context to query the collections in the database, which you can do by also changing the resolvers in the file `src/index.js`:

```js
// src/index.js

...

// Provide resolver functions for your schema fields
const resolvers = {
  events: async (_, context) => {
    const { db } = await context();
    return db
      .collection('events')
      .find()
      .toArray();
  },
  event: async ({ id }, context) => {
    const { db } = await context();
    return db.collection('events').findOne({ id });
  },
};

...

```

Your resolvers are now using the `db` from the context to send a query to the MongoDB mock server. The first resolver is returning all the events whenever you use the `events` query, as described in your schema. The second resolver isn't only using the context from the request, but also the argument `id` that's already destructured from the arguments object.

In the next section, you'll test if these queries are working as expected by using the GraphQL Playground interface instead of a cUrl request from your terminal.

### Query a GraphQL server

As your resolvers are now connected to the mock MongoDB instance, you should be able to send queries to the GraphQL server. Instead of using the terminal to send documents to the GraphQL server, you can also install a more visual interface to help you do this. Out-of-box `express-graphql` middleware offers the tool GraphiQL as a solution, but that one lacks some of the features you'll be needing further on in this tutorial. Therefore the tool [`graphql-playground-middleware-express`](https://www.npmjs.com/package/graphql-playground-middleware-express) will be used instead, which you can install from npm using the command:

```
npm install graphql-playground-middleware-express
```

When the installation process has finished, this interface can be added to the setup by adding the following to the file `src/index.js`:

```js
// src/index.js
const express = require('express');
const graphqlHTTP = require('express-graphql');
const schema = require('./schema');
const startDatabase = require('./database');
const expressPlayground =
	require('graphql-playground-middleware-express').default;

...

const app = express();
app.use(
  '/graphql',
  graphqlHTTP({
    schema,
    rootValue: resolvers,
    context,
  }),
);
app.get('/playground', expressPlayground({ endpoint: '/graphql' }));
app.listen(4000);

console.log(`🚀 Server ready at http://localhost:4000/graphql`);
```

You import the default export of the `graphql-playground-middleware-express` module and create a `GET /playground` endpoint in your Express app to serve the GraphQL playground environment.

If you visit the URL [`http://localhost:4000/playground`](http://localhost:4000/playground) in your browser now, you'll see something like the image below.

![GraphQL Playground interface](https://cdn.auth0.com/blog/building-and-securing-a-graphql-server-with-nodejs/graphql-playground-interface.png)

> If this page isn't available, ensure that your GraphQL server is still running by checking the terminal. As you might remember, the command to start the server is `npm start`.

To test if the data is successfully retrieved from your mock database, you can send the document below to your GraphQL server using the Playground by copy-pasting this in the left side of the page and then clicking on the circular "play" button:

```graphql
query {
  events {
    title
    date
    attendants {
      name
    }
  }
}
```

This should return the following data that will contain the information from the database about the events, and in the same structure as you've defined in your query:

```json
{
  "data": {
    "events": [
      {
        "title": "GraphQL Introduction Night",
        "date": "2019-11-06T17:34:25+00:00",
        "attendants": [
          {
            "name": "Peter"
          },
          {
            "name": "Kassandra"
          }
        ]
      },
      {
        "title": "GraphQL Introduction Night #2",
        "date": "2019-11-06T17:34:25+00:00",
        "attendants": [
          {
            "name": "Kim"
          }
        ]
      }
    ]
  }
}
```

You can play around with this query and delete or add fields however you please, as that's the great advantage of the flexible structure of GraphQL. To retrieve information about a single event, you can send the following document that contains the query to retrieve the event and its data. Of course, you can add whatever fields you like, as long as they're mentioned in the schema.

```graphql
query {
  event(id: 1) {
    title
    date
    attendants {
      name
    }
  }
}
```

The output of the query above can be again seen in the GraphiQL interface, and from the query, you can already predict what structure the result will follow. Besides sending queries, you probably also want to mutate data in your database, which will be demonstrated in the next section.

### Mutating data with GraphQL

In this section, you'll also make it possible to send mutations to the server that lets you change the details of an event like the title and description. Mutations need to be added to the schema, similarly to the queries. To add the mutation to your schema, you need to make the following changes to the file `src/schema.js`:

```js
// src/schema.js

const { buildSchema } = require('graphql');

// Construct a schema, using GraphQL schema language
const schema = buildSchema(`
    type Event {
        id: ID!
        title: String!
        description: String
        date: String
        attendants: [Person!]
    }

    type Person {
        id: ID!
        name: String!
        age: Int
    }

    type Query {
        events: [Event!]!
        event(id: Int!): Event!
    }

    type Mutation {
      editEvent(id: Int!, title: String!, description: String!): Event!
    }
`);

module.exports = schema;
```

After adding the mutation to the schema, the resolvers also need to be able to handle this mutation by adding another resolver that can edit the data in the mock database. This mutation takes the arguments `id`, `title`, and `description`, and these can, therefore, also be used in the resolvers to mutate the data in the database. To make the mutation work change the resolver in the file `src/index.js` as follows:

```js
// src/index.js

...

// Provide resolver functions for your schema fields
const resolvers = {
  events: async (_, context) => {
    const { db } = await context();
    return db
      .collection('events')
      .find()
      .toArray();
  },
  event: async ({ id }, context) => {
    const { db } = await context();
    return db.collection('events').findOne({ id });
  },
 editEvent: async ({ id, title, description }, context) => {
   const { db } = await context();

   return db
     .collection('events')
     .findOneAndUpdate(
       { id },
       { $set: { title, description } },
       { returnOriginal: false },
     )
     .then(resp => resp.value);
 },
};

...

```

This resolver looks at the mutation `addEvent` and the arguments as described in the schema, after updating the data in the mock database, the new data will be returned. You can test this mutation by using the GraphQL Playground and send the following mutation from it:

```graphql
mutation {
  editEvent(
    id: 2
    title: "Something else"
    description: "New information about this event"
  ) {
    title
    description
  }
}
```

The output of this mutation will be the new information for the event, as the mutation is linked to the type `Event`:

```json
{
  "data": {
    "editEvent": {
      "title": "Something else",
      "description": "New information about this event"
    }
  }
}
```

Next to `title` and `description`, you can also request other fields from that type in this mutation. As you probably don't want everyone to be able to change the details for this event, you'll learn how to add authentication and authorization to your GraphQL server in the next section.

{% include tweet_quote.html quote_text="Learn how to test GraphQL queries using the GraphQL Playground interface instead of using cUrl requests." %}

## Securing a GraphQL Server with Auth0

So far, you've already learned how to create a GraphQL server with Node.js, and send queries to it that retrieve data from a mock MongoDB instance. Also, users can mutate an event or subscribe to one. The only thing missing here is authentication and authorization, for example, to secure the usage of mutations or limit the amount of information that's returned by a query.

### Validating JWTs on the GraphQL server

The prerequisites of this tutorial stated that you need an Auth0 account to complete all the sections, which you can still get by <a href="https://auth0.com/signup" data-amp-replace="CLIENT_ID" data-amp-addparams="anonId=CLIENT_ID(cid-scope-cookie-fallback-name)">signing up for a free Auth0 account here</a>, if you haven't created it already.

Once you have an account, you can proceed by going to the [Auth0 dashboard](https://manage.auth0.com/#/apis), where you should click on the _Create API_ button. You'll be redirected to a new page where you can create an API that's able to issue and validate JSON Web Tokens (JWTs).

The form to create a new API requires the following information:

- Name: This name will be used to display your API in the Auth0 Dashboard, e.g., you can use something like "Auth0 GraphQL Server Tutorial".

- Identifier: An URL-like identifier for the API that you're creating, you can fill something like `https://graphql-api` in here. This URL will never be called by Auth0.

- Signing Algorithm: Leave this set to `RS256` as this is the recommended one.

![Auth0 form used to create a new API](https://cdn.auth0.com/blog/developing-a-secure-api-with-nestjs/auth0-dashboard-new-api-form.png)

After adding this information to the form, you can click the _Create_ button. You are then taken to the "Quick Start" section of the Auth0 API you just created. In here, click on the "Node.js" tab to get an idea of the code you'll need to use in your GraphQL server to validate JWTs issued by Auth0.

There are two critical values on the Node.js code snippet that you'll need in a few moments: the values of the `audience` and `issuer` properties of the object argument passed to the `jwt` function.

Leave this "Quick Start" page open and return to the project in your terminal where you'd need to run the following command:

```
npm install dotenv jsonwebtoken jwks-rsa
```

This command will install three packages: [`dotenv`](https://www.npmjs.com/package/dotenv) is used to store local environment variables, [`jsonwebtoken`](https://www.npmjs.com/package/jsonwebtoken) and [`jwks-rsa`](https://www.npmjs.com/package/jwks-rsa) are needed to check if a JWT is valid.

First, you need to create a new file in the directory `src` that's called `validate.js`:

```bash
touch src/validate.js
```

This file will hold the code to validate the JWTs that are sent to your GraphQL server. In this file, you need to add the following code:

```js
// src/validate.js

require('dotenv').config();
const jwt = require('jsonwebtoken');
const jwksClient = require('jwks-rsa');

const client = jwksClient({
  jwksUri: `https://${process.env.AUTH0_DOMAIN}/.well-known/jwks.json`,
});

function getKey(header, callback) {
  client.getSigningKey(header.kid, function(error, key) {
    const signingKey = key.publicKey || key.rsaPublicKey;
    callback(null, signingKey);
  });
}

async function isTokenValid(token) {
  if (token) {
    const bearerToken = token.split(' ');

    const result = new Promise((resolve, reject) => {
      jwt.verify(
        bearerToken[1],
        getKey,
        {
          audience: process.env.API_IDENTIFIER,
          issuer: `https://${process.env.AUTH0_DOMAIN}/`,
          algorithms: ['RS256'],
        },
        (error, decoded) => {
          if (error) {
            resolve({ error });
          }
          if (decoded) {
            resolve({ decoded });
          }
        },
      );
    });

    return result;
  }

  return { error: 'No token provided' };
}

module.exports = isTokenValid;
```

The function `isTokenValid` in the file above checks the validity of a JWT by using the `verify` function from the library `jsonwebtoken`, which uses your `AUTH0_DOMAIN` and `API_IDENTIFIER` values from Auth0. These values are loaded from the `process.env` variable with `dotenv` and are used to create a `jwksClient` with the library `jwks-rsa`. You'll be creating a `.env` file to store these values soon.

If the token send to `isTokenValid` can be validated, the promise will resolve and the token will be marked as valid. Otherwise, the promise will return an error, which also happens when no token is provided. Note that you don't reject the promise as the error handling is done in the resolvers later on.

The values for these constants need to be added to a hidden file `.env`, making it the place to store sensitive values locally. Ensure to be in the root folder of the project and create the `.env` file by running this command:

```bash
touch .env
```

Now, place the code block below in that file:

```
AUTH0_DOMAIN=YOUR_AUTH0_DOMAIN
API_IDENTIFIER=YOUR_API_IDENTIFIER
```

The values of `YOUR_AUTH0_DOMAIN` and `YOUR_API_IDENTIFIER` must be replaced by the values from your Auth0 API "Quick Start" page as follows:

- The value of `AUTH0_DOMAIN` is the value of the `issuer` object property from the code snippet, without the protocol, `https://`, the quotation marks, and the trailing slash. It follows this format `YOUR-AUTH0-TENANT.auth0.com`.

- The value of `API_IDENTIFIER` is the value of the `audience` object property from the code snippet, and it's the same value that you provided as an identifier to your Auth0 API earlier on. Do not include quotation marks.

> If you're using git for the version control of your code, make sure that the file `.env` is listed in your `.gitignore` file. Otherwise, your local environment variables might be exposed to the world.

Your server is now able to validate JWTs, but you haven't been able to read them from the GraphQL documents yet. How to request and pass tokens to the validation function is demonstrated in the next section of this tutorial.

### Requesting and passing JWT to the GraphQL Server

The validation function can validate JWTs that are created by Auth0 and there are a lot of ways to create tokens with Auth0. This tutorial won't focus on how to receive or sign a token, but if you want more information on this, have a look at the [_Login with Auth0_ document](https://auth0.com/docs/login).

From the [Auth0 Dashboard](https://manage.auth0.com/#/apis), you're able to see the list of APIs that you've linked to your account. If you click on the API that you've created for this tutorial (e.g., "Auth0 GraphQL Server Tutorial"), you can find a test JWT on the tab _Test_, under the section labeled _Response_. The block with the test token has the following format:

```json
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbG...p4TkQ",
  "token_type": "Bearer"
}
```

To construct a JWT from this block, you need to combine the fields `access_token` and `token_type` to make this token as follows:

```
Bearer eyJ0eXAiOiJKV1QiLCJhbG...p4TkQ
```

You can send this token with your request to the GraphQL server by including it in the `Authorization` header, or by using the GraphQL Playground.

If you open the Playground, you can see two tabs in the bottom left side of the interface, where one has the label _Query Variables_ and the other _HTTP Headers_. As you might figure, the last one can be used to send headers with your GraphQL document. These headers can be added in a JSON format as seen below:

```json
{
  "Authorization": "Bearer eyJ0eXAiOiJKV1QiLCJhbG...p4TkQ"
}
```

If you now send a new request to the GraphQL server, nothing will happen, as the token isn't received by the server just yet. Making it possible to read the token from your server will require some changes to `src/index.js`: the `context` must be able to read the information from your request by changing the setup of `graphqlHTTP`. Make sure to add the parenthesis here to the `context` field of `graphqlHTTP` as you need to have a callback here that calls the `context` function:

```js
// src/index.js

...


const app = express();
app.use(
  '/graphql',
  graphqlHTTP(async (req) => ({
    schema,
    rootValue: resolvers,
    context: () => context(req)
  })),
);
app.get('/playground', expressPlayground({ endpoint: '/graphql' }));
app.listen(4000);

console.log(`🚀 Server ready at http://localhost:4000/graphql`);
```

With the change above, your `context` function will now receive the request information whenever you send a document to the GraphQL server. In your context function, you can get the token from the request headers by changing the `context` function below and importing the `isTokenValid` function:

```js
// src/index.js
const express = require('express');
const graphqlHTTP = require('express-graphql');
const schema = require('./schema');
const startDatabase = require('./database');
const expressPlayground =
	require('graphql-playground-middleware-express').default;
const isTokenValid = require('./validate');

// Create a context for holding contextual data
const context = async req => {
  const db = await startDatabase();
  const { authorization: token } = req.headers;

  return { db, token };
};

// Provide resolver functions for your schema fields
const resolvers = {

...

```

Finally, you need to read the token from the context object in your resolvers. As you don't want users to change events without a valid token, the `isTokenValid` function must be called with the token in the resolver for your `editEvent` mutation:

```js
// src/index.js

...

// Provide resolver functions for your schema fields
const resolvers = {

  ...

  editEvent: async ({ id, title, description }, context) => {
    const { db, token } = await context();

    const { error } = await isTokenValid(token);

    if (error) {
      throw new Error(error);
    }

    return db
      .collection('events')
      .findOneAndUpdate(
        { id },
        { $set: { title, description } },
        { returnOriginal: false },
      )
      .then(resp => resp.value);
  },

  ...

```

You can check the result of the token validator by sending the mutation for the event as you've already done before with and without the headers. If you provided the header with the test token from Auth0, you're able to execute the mutation, while failing to include a token, or including an invalid one, will return the following error:

```json
{
  "error": {
    "errors": [
      {
        "message": "JsonWebTokenError: invalid token",
        "locations": [
          {
            "line": 2,
            "column": 3
          }
        ],
        "path": ["editEvent"]
      }
    ],
    "data": null
  }
}
```

The error message above will be displayed when you send an invalid token with your document. If you don't send any token at all for this mutation, the message _"No token provided will be displayed."_

Include the authorization header and send the following document from the playground:

```graphql
mutation {
  editEvent(
    id: 2
    title: "New title"
    description: "Newer information about this event"
  ) {
    title
    description
  }
}
```

This time around, you should get a successful response from the server.

Just as you want users to be authenticated to edit events, you might also want users to be specifically authorized to view certain information from your queries. How to handle this for your GraphQL server will be handled in the next section.

{% include tweet_quote.html quote_text="Learn how to handle authentication and authorization of a GraphQL server using Node.js and JWTs." %}

### Limiting responses

Some information in your queries might be private and only intended for users with a valid token, as you've already applied to the mutation `addEvent`. Assume you don't want every user to be able to see which people are attending an event. Then you need to check for a valid token and otherwise, don't return any information for the field `attendants`. The way GraphQL would handle such a scenario is by nullability. Instead of throwing an error, the data will be set to `null` and, therefore, can't be read.

Suppose you want to limit the data that's returned by the query for a list of events (let's leave out a single event for now), then you would need to send a valid token with your document to see the list of attendants. To accomplish this, you can use the same `isTokenValid` function from the previous section, but instead of throwing an error, you would be limiting the fields that are queried on the MongoDB database. You can do this by making the following changes to `src/index.js`:

```js
// src/index.js

...

// Provide resolver functions for your schema fields
const resolvers = {
  events: async (_, context) => {
    const { db, token } = await context();
    const { error } = await isTokenValid(token);
    const events = db.collection("events").find();
    return !error
      ? events.toArray()
      : events.project({ attendants: null }).toArray();
  },
  event: async ({ id }, context) => {
    const { db, token } = await context();
    const { error } = await isTokenValid(token);
    const event = await db.collection("events").findOne({ id });
    return !error ? event : { ...event, attendants: null };
  },

  ...

```

With this last change, the field `attendants` can no longer be requested by users that don't add a JWT to their requests headers. Try this out by sending the following document to the server but without including the authorization header:

```graphql
query {
  events {
    title
    date
    attendants {
      name
    }
  }
}
```

You won't get an error as a response. Instead, you'll get a response object that has an `attendants` field set to `null`. The `attendants` field will also be `null` when you query the database for a single event.

## Conclusion

In this tutorial, you've created a GraphQL server with Node.js, that uses a mock MongoDB instance for its data and Auth0 for authorization and authentication. You've learned how to set up a basic server with Express, define a GraphQL schema, and write resolvers that can collect data from a MongoDB database. Basic authentication is added by using validation JWTs that are issued by your Auth0 account, making it possible to secure certain queries and mutations. The full code for this tutorial can be found on [Github](https://github.com/auth0-blog/auth0-graphql-server).

You can go further with learning how to handle authorization by implementing a Role-Based Access Control in your API; however, getting a token with roles is much easier if you have a working client, which I’ll teach you how to build in an upcoming tutorial.
