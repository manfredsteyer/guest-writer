---
layout: post
title: Developing Secure Web Applications with React and GraphQL
metatitle: Developing Secure Web Applications with React and GraphQL
description: Develop a secure web application that handles authentication and authorization using React and GraphQL
metadescription: Develop a secure web application that handles authentication and authorization using React and GraphQL
date: 2020-01-30 10:30
category: Developers, Tutorial, GraphQL
post_length:
community_topic_id: 
author:
  name: Roy Derks
  url: https://twitter.com/gethackteam
  avatar: https://cdn.auth0.com/blog/guest-authors/roy-derks.png
design:
  illustration: https://cdn.auth0.com/blog/illustrations/ambassador-in-the-moon.png
tags:
  - react
  - graphql
  - apollo
related:
- 
---

**TL;DR:** The introduction of GraphQL in 2015 changed how applications can request information from an API, and how that information flows through these applications. No longer do you have to filter large chunks of data to display the correct information to your users, but instead, by using GraphQL, you can specifically request whatever information you want to show them. React is a great library to create web applications with ease and handle side effects like authentication using the lifecycles it provides.

This post will show how you can query and mutate data from a GraphQL server, display this information in a React application, and secure this application using Auth0.

## Prerequisites

Before you continue reading this tutorial, you'll need to make sure that you have `node` and `npm` installed on your machine. If you don't have these installed yet, you can find the installation instructions [here](https://nodejs.org/en/download/).

This post will be using a GraphQL server that is set up to handle authentication and authorization with Auth0. The code for this server can be found [here](https://github.com/auth0-blog/auth0-graphql-server). To run the server, you need to follow the steps in the _Getting started_ section README of that project, including [adding your Auth0 information](https://auth0.com/blog/build-and-secure-a-graphql-server-with-node-js/#Securing-a-GraphQL-Server-with-Auth0). If you don't have an Auth0 account yet, you can <a href="https://auth0.com/signup" data-amp-replace="CLIENT_ID" data-amp-addparams="anonId=CLIENT_ID(cid-scope-cookie-fallback-name)">register one for free here</a>. Also, I'm assuming you already have prior knowledge about JavaScript and React and know how you can create components and update state.

> **Note** You will need write down the values for `AUTH0_DOMAIN` and `API_IDENTIFIER` from the `.env` file of the GraphQL server, as you'll also need these values for this post.

## What is GraphQL?

In short, GraphQL is a query language for APIs that lets you query and mutate data over HTTP. It was publicly released by Facebook in 2015 to help them provide their mobile applications with just the data they need instead of sending "raw" data over REST that needs to be filtered in the frontend. If you feel like you need to learn more about GraphQL, you can read a longer description in the [first part](https://github.com/auth0-blog/auth0-graphql-server) of this series about GraphQL.

## Developing Secure Web Applications with React and GraphQL

Web applications have rapidly increased in complexity as more and more technologies and tools for the web are available nowadays. One of these technologies is React, a JavaScript library for creating Single-Page Applications (SPA) or mobile applications. Another technology that has arisen the past few years is GraphQL, which simplifies how you can query and mutate data over HTTP. When used together, these technologies let you create future-proof web applications that can be secured using Auth0. In this post, you'll learn how to develop a secure web application with React and GraphQL using Apollo. Not only will this application handle authentication, logged in users can have different authorization levels that define the actions they can take in the application.

## Creating a SPA with React

In this first section, you'll set up a basic React application with [Create React App](https://github.com/facebook/create-react-app). As mentioned before, React is a library that helps you create modern web applications and provides you with building blocks to handle more complex concepts like state management.

### Getting started with React

To render React in your browser, you need to compile it first. For this, you need to set up a module builder like Webpack and something to compile your JavaScript code like Babel. However, when you create a new React application with Create React App, you no longer need to configure these manually. You can install Create React App globally on your machine and initiate a new project by running the following command in your terminal:

```
npx create-react-app auth-graphql-app
```

This command will start the installation process of Create React App and create a new directory called `auth-graphql-app` that holds the boilerplate code for your React application. You can move into this directory and start the application by running:

```
cd auth-graphql-app && npm start
```

The command `npm start` will run the start script of Create React App to compile and run your code on a local development server, which you can see by opening [http://localhost:3000](http://localhost:3000) in your browser. On this page, you'll see the first version of your application. The application, at this point, will look like this:

![create react app](articles/initial-create-react-app.jpg)

The application you'll build in this post will display a list of events and allow you to manage the attendants of those events. It will consist of two pages: the page to list the events and a page to edit the events. Before you connect with the GraphQL server and create new components to display the responses from the server, you need to know how the Create React App project is structured. Let's start with the file `src/index.js`, which is the entry point of your React application. In this file, a component called `App` is rendered by `ReactDOM`.

This means that all the components that you'd like to render in the browser must be reachable from that component that's defined in the file `src/App.js`, including routing. Routing in React (or other Single-Page Applications) works differently from routing in regular websites and requires the usage of a library called `react-router-dom`. This package can be installed from npm by running the command below in your terminal:

```
npm install react-router-dom
```

With `react-router-dom`, you can import a `Router` component that will be used by your application to dynamically change between pages without having to refresh the page. This `Router` component must be added at the top level in `src/index.js` and it must wrap the `App` component:

```js
// src/index.js

import React from 'react';
import ReactDOM from 'react-dom';
import { BrowserRouter as Router } from 'react-router-dom';
import App from './App';
import * as serviceWorker from './serviceWorker';

ReactDOM.render(
  <Router>
    <App />
  </Router>,
  document.getElementById('root')
);

...

```

The routes for your application can be defined in the `src/App.js` file. Delete all the code that's currently in this file and replace it with the following:

```js
import React from 'react';
import { Switch, Route } from 'react-router-dom';

function App() {
  return (
    <Switch>
      <Route path='/event/:id'>Event</Route>
      <Route path='*'>All Events</Route>
    </Switch>
  );
}

export default App;
```

After changing the file above, you've created two routes for your application: `/event/:id` and `*`. The `Switch` component first tries to find the route for a single event and, if it's not found, directs to the _All Events_ route. The route for a single event can be found by visiting a page with the url `http://localhost:3000/event` followed by the event `id`, e.g: [`http://localhost:3000/event/123`](http://localhost:3000/event/123). Later on in this tutorial, you'll add the data to these routes.

Next, you can delete the following files from the project as you will no longer be needing these:

```
src
|-- App.css
|-- App.test.js
|-- index.css
|-- logo.svg
|-- setupTests.js
```

The project is now ready to be connected to the GraphQL server, which will use [Apollo](https://www.apollographql.com/docs/react/) in the next section.

## Using GraphQL with Apollo

The data for the application you create in this post is returned by the GraphQL server that was described in the _Prerequisites_ section. Setting up this GraphQL server is required to continue with the steps in the remainer of this post. 

### Setting Up a GraphQL Server
To set up the GraphQL server that is needed to get the data for the React application that you've built so far, you need to follow all the steps in the tutorial [Build and Secure a GraphQL Server with Node.js](https://auth0.com/blog/build-and-secure-a-graphql-server-with-node-js/) or clone the project from [Github](https://github.com/auth0-blog/auth0-graphql-server). 

If you've completed the tutorial to create a GraphQL server, you can proceed to the next step _Set up Apollo client with React_. Otherwise, after cloning the repository you need to move into the newly created directory and follow the steps in the _Getting started_ section of the readme. This means you need to create a new file called `.env`:

```bash
touch .env
```
And add the following values to that file.

```
AUTH0_DOMAIN=YOUR_AUTH0_DOMAIN
API_IDENTIFIER=GRAPHQL_SERVER_API_IDENTIFIER
```

The values of `YOUR_AUTH0_DOMAIN` and `GRAPHQL_SERVER_API_IDENTIFIER` must be replaced by the values from your Auth0 account:

- The value of `AUTH0_DOMAIN` is the value of the issuer object property from the code snippet, without the protocol, https://, the quotation marks, and the trailing slash. It follows this format: `YOUR-AUTH0-TENANT.auth0.com`.

- The value of `API_IDENTIFIER` is the value of the audience object property for the GraphQL server, that you must create on the [API section](https://manage.auth0.com/#/apis) of the Auth0 dasboard. Do not include quotation marks.

After creating this file and adding the correct values to it, you can install and start the GraphQL server by running:

```bash
npm install && npm start
```

The GraphQL server will become available at [`http://localhost:4000/graphql`](http://localhost:4000/graphql). You can also see an interactive playground at [`http://localhost:4000/playground`](http://localhost:4000/playground). Using this GraphQL Playground interface, you can inspect the schema of this server or send documents containing queries and/or mutations to it. An example of a document with a query that can be handled by this GraphQL server is:

```
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

The response of this query will be the full list events including its `title`, `date`, and the `name` of every attendant of the event. This response will be in JSON and looks like the following:

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

As mentioned before, you can send documents to the GraphQL server over plain HTTP, but also by using a package like [Apollo](https://www.apollographql.com/docs/react/). With Apollo, you can connect with the GraphQL server, handle sending documents to the server, and enable caching for data retrieved from the GraphQL server.

### Set up Apollo client with React

Before you can send documents to the GraphQL server, you need to set up the connection with the server, which can be done by adding the package `apollo-boost`. This package configures a GraphQL client for your application with Apollo's recommended settings for caching, state management, and error handling. Next to `apollo-boost`, you also need to install the packages `@apollo/react-hooks` and `graphql`. With `@apollo/react-hooks`, you can handle queries and mutations from your React components, while `graphql` is needed to use GraphQL's query language inside your application. To add these packages to your application, run the following command from your terminal:

```
npm install apollo-boost @apollo/react-hooks graphql
```

In the file `src/index.js`, you can subsequently import the method `ApolloClient` from [`apollo-boost`](https://github.com/apollographql/apollo-client/tree/master/packages/apollo-boost) that will create the GraphQL client for you. This method needs the endpoint to the GraphQL server that is running on `http://localhost:4000/` to get you started. By default, the client expects the GraphQL server to be running on the `/graphql` endpoint. Creating the client is done by making the changes below to `src/index.js`:

```js
// src/index.js
import React from 'react';
import ReactDOM from 'react-dom';
import { BrowserRouter as Router } from 'react-router-dom';
import ApolloClient from 'apollo-boost';
import App from './App';
import * as serviceWorker from './serviceWorker';

const client = new ApolloClient({
  uri: 'https://48p1r2roz4.sse.codesandbox.io'
});

ReactDOM.render(
  <Router>
    <App />
  </Router>,
  document.getElementById('root')
);

...

```

But creating a client isn't sufficient to connect your React application to the GraphQL server, as you also need to create a Provider that wraps your application and makes it possible to access the client from everywhere in your project. This Provider uses the Context API from React. I [previously wrote a blog post](https://auth0.com/blog/handling-authentication-in-react-with-context-and-hooks/) about it, so you can check that out if you aren't familiar with it.

You can create a Provider by importing `ApolloProvider` from `@apollo/react-hooks` in the file `src/index.js` and pass the client to it. The `ApolloProvider` must be placed inside the `Router` to wrap the `App` component that is rendered by `ReactDOM` by making the following changes:

```js
// src/index.js
import React from 'react';
import ReactDOM from 'react-dom';
import { BrowserRouter as Router } from 'react-router-dom';
import ApolloClient from 'apollo-boost';
import ApolloProvider from '@apollo/react-hooks';
import App from './App';
import * as serviceWorker from './serviceWorker';

const client = new ApolloClient({
  uri: 'http://localhost:4000',
});

ReactDOM.render(
  <Router>
    <ApolloProvider client={client}>
      <App />
    </ApolloProvider>
  </Router>,
  document.getElementById('root'),
);
```

From any component that's nested within `ApolloProvider`, you can now send documents with queries or mutations using `@apollo/react-hooks`, which will be demonstrated in the next part of this section.

### Query the GraphQL server with Apollo

By wrapping the `App` component with `ApolloProvider` you can now use the methods `useQuery` and `useMutation` from `@apollo/react-hooks` to request and mutate data from the GraphQL server. These methods use the [Hooks pattern](https://www.apollographql.com/docs/react/api/react-hooks/) of React and take a GraphQL document as a parameter.

First, let's add a list of all the events to the application by sending a document with a query to the GraphQL server using `useQuery`. This should be done from a new React component. In the directory `src`, create a new file called `Events.js` with the following command:

```bash
touch src/Events.js
```

The query for getting the events has been mentioned in this post before and must be added to this file together with the `useQuery` Hook:

```js
// src/Events.js
import React from 'react';
import { gql } from 'apollo-boost';
import { useQuery } from '@apollo/react-hooks';

const GET_EVENTS = gql`
  query getEvents {
    events {
      id
      title
      date
    }
  }
`;

function Events() {
  const { loading, data, error } = useQuery(GET_EVENTS);

  if (loading) return 'Loading...';
  if (error) return 'Something went wrong...';

  return (
    <ul>
      {data.events.map(({ id, title, date }) => (
        <li key={id}>
          {title} ({date})
        </li>
      ))}
    </ul>
  );
}

export default Events;
```

The query `getEvents` is passed to the `useQuery` Hook that returns the variable `loading` when it gets called. After resolving, the Hook will return either a `data` object with the events or an `error` object. If all goes well, the `data` object contains an array with the event which you can iterate over to return a list of all the events.

To see the list of events in your browser, you also need to import the `Events` component in the file `src/App.js` and have it returned by the `*` routes in your `App` component. Also, let's add a header with a title to this component with some styling. Styling can be done in several ways with React, and one of the easiest ways to add styling to a React component is by using inline style attributes that require properties to be written in camel case. Let's see how this works by adding inline styling to the `App` component:

```js
// src/App.js
import React from 'react';
import { Switch, Route } from 'react-router-dom';
import Events from './Events';

function App() {
  return (
    <div style={{ fontFamily: 'Helvetica' }}>
      <header
        style={{
          display: 'inline-block',
          width: '100%',
          backgroundColor: 'lightBlue',
          padding: '10 20px',
          textAlign: 'center',
          borderRadius: '5px',
        }}
      >
        <h1 style={{ color: 'white' }}>Events</h1>
      </header>
      <Switch>
        <Route path='/event/:id'>Event</Route>
        <Route path='*'>
          <Events />
        </Route>
      </Switch>
    </div>
  );
}

export default App;
```

If you open the browser again, you can see that the list of events is being displayed together with the `title` and `date` of each event. This page could also use some styling, which can be done by changing the following in the file `src/Events.js`:

```js
// src/Events.js

...

function Events() {
  const { loading, data, error } = useQuery(GET_EVENTS);

  if (loading) return 'Loading...';
  if (error) return 'Something went wrong...';

  return (
    <ul style={{ listStyle: 'none', width: '100%', padding: '0' }}>
      {data.events.map(({ id, title, date }) => (
        <li
          key={id}
          style={{
            backgroundColor: 'lightGrey',
            marginBottom: '10px',
            padding: '10px',
            borderRadius: '5px',
          }}
        >
          <h2>{title}</h2>
          <span style={{ fontStyle: 'italic' }}>{date}</span>
        </li>
      ))}
    </ul>
  );
}

export default Events;
```

But having just a list of all the events is not sufficient, as you also want to display the attendants of that event. However, this information is private and requires you to send a valid JWT along with your query. This JWT can be retrieved by sending a request to the Auth0 authentication service, which you'll do in the next section of this post.

## Securing a React app

Auth0 is used to secure your React application, just as it was used to secure the GraphQL server that you're using for this post. By sending a request to the Auth0 authentication service with your credentials, you'll retrieve a JWT that can be validated by the GraphQL server.

### Handle Authentication

Sending a request to Auth0 can be done by using the package [`@auth0/auth0-spa-js`](https://auth0.com/docs/quickstart/spa/react/01-login) together with your Auth0 `Domain` and `Client ID`. To get these values you need to create a new Single-Page Application on the [Application Settings](https://manage.auth0.com/#/applications/) page in the Auth0 dashboard. Make sure to use the same Auth0 account as the one you used to set up the GraphQL server. To proceed, you need to install both `@auth0/auth0-spa-js` from npm:

```
npm install @auth0/auth0-spa-js
```

After installing this package, you must add a new file called `src/react-auth0-spa.js` in your project by running:

```bash
touch src/react-auth0-spa.js
```

And place the following code inside:

```js
// src/react-auth0-spa.js
import React, { useState, useEffect, useContext } from 'react';
import createAuth0Client from '@auth0/auth0-spa-js';

const DEFAULT_REDIRECT_CALLBACK = () =>
  window.history.replaceState({}, document.title, window.location.pathname);

export const Auth0Context = React.createContext();
export const useAuth0 = () => useContext(Auth0Context);
export const Auth0Provider = ({
  children,
  onRedirectCallback = DEFAULT_REDIRECT_CALLBACK,
  ...initOptions,
}) => {
  const [isAuthenticated, setIsAuthenticated] = useState();
  const [user, setUser] = useState();
  const [auth0Client, setAuth0] = useState();
  const [loading, setLoading] = useState(true);
  const [popupOpen, setPopupOpen] = useState(false);

  useEffect(() => {
    const initAuth0 = async () => {
      const auth0FromHook = await createAuth0Client(initOptions);
      setAuth0(auth0FromHook);

      if (window.location.search.includes('code=')) {
        const { appState } = await auth0FromHook.handleRedirectCallback();
        onRedirectCallback(appState);
      }

      const isAuthenticated = await auth0FromHook.isAuthenticated();

      setIsAuthenticated(isAuthenticated);

      if (isAuthenticated) {
        const user = await auth0FromHook.getUser();
        setUser(user);
      }

      setLoading(false);
    };
    initAuth0();
    // eslint-disable-next-line
  }, []);

  const loginWithPopup = async (params = {}) => {
    setPopupOpen(true);
    try {
      await auth0Client.loginWithPopup(params);
    } catch (error) {
      console.error(error);
    } finally {
      setPopupOpen(false);
    }
    const user = await auth0Client.getUser();
    setUser(user);
    setIsAuthenticated(true);
  };

  const handleRedirectCallback = async () => {
    setLoading(true);
    await auth0Client.handleRedirectCallback();
    const user = await auth0Client.getUser();
    setLoading(false);
    setIsAuthenticated(true);
    setUser(user);
  };
  return (
    <Auth0Context.Provider
      value={{
        isAuthenticated,
        user,
        loading,
        popupOpen,
        loginWithPopup,
        handleRedirectCallback,
        getIdTokenClaims: (...p) => auth0Client.getIdTokenClaims(...p),
        loginWithRedirect: (...p) => auth0Client.loginWithRedirect(...p),
        getTokenSilently: (...p) => auth0Client.getTokenSilently(...p),
        getTokenWithPopup: (...p) => auth0Client.getTokenWithPopup(...p),
        logout: (...p) => auth0Client.logout(...p),
      }}
    >
      {children}
    </Auth0Context.Provider>
  );
};
```

This file sets up the connection with Auth0 and returns a Provider called `Auth0Provider` and a Hook. This Provider is similar to `ApolloProvider` and needs your Auth0 credentials, making it possible to use the `useAuth0` Hook to connect with Auth0 from any components that are nested inside.

Before adding the `Auth0Provider` to your project you need to store the `Domain` and `Client ID` somewhere safe, like a local environment file. When you create an application using Create React App, you can create a `.env` file in the root folder of your project and use the constants in this file from the `process.env` variable. Also, you'll be needing the Auth0 details from the GraphQL server as these are needed to create a JWT that can be validated by the GraphQL server.

> **Note:** These constants need to be prefixed with `REACT_APP_` so they can be used inside an application created with Create React App.

From the root directory of the project, run this command to create the `.env` file:

```bash
touch .env
```

Place the following code in this file:

```
REACT_APP_AUTH0_DOMAIN=YOUR_AUTH0_DOMAIN
REACT_APP_CLIENT_ID=YOUR_CLIENT_ID
REACT_APP_API_IDENTIFIER=GRAPHQL_SERVER_API_IDENTIFIER
```

The values of `YOUR_AUTH0_DOMAIN` and `YOUR_CLIENT_ID` must be replaced by the values from your Auth0 React "Quick Start" page as follows:

- The value of `AUTH0_DOMAIN` is the value of the `issuer` object property from the code snippet, without the protocol, `https://`, the quotation marks, and the trailing slash. It follows this format: `YOUR-AUTH0-TENANT.auth0.com`.

- The value of `CLIENT_ID` is a unique public identifier for your application. Although it's a public identifier, it’s recommended to not make it easily guessable by third parties.

As mentioned before the value of `API_IDENTIFIER` must be the same value as you used when creating the GraphQL server, this value can be found in the [API section](https://manage.auth0.com/#/apis) of the Auth0 dasboard:

- The value of `API_IDENTIFIER` is the value of the audience object property for the GraphQL server, and it's the same value that you provided as an identifier to your Auth0 API earlier on. Do not include quotation marks.

Next to these credentials, you need to set the _Callback URL_, _Logout URL_ and _Allowed Web Origins_ for your application on the [Application Settings](https://manage.auth0.com/#/applications/) page in the Auth0 dashboard. These values must be equal to the address where your React application is running, which is `http://localhost:3000`.

You can now restart the development server of Create React App by running `npm start` again. Your Auth0 credentials should now be available to use in the application to set up `Auth0Provider` in the file `src/index.js` and be placed around the `ApolloProvider` component. Also, `Auth0Provider` should call a callback function to redirect the user to the correct page after authentication. This function is used to clear the user's authentication code from the url to prevent them from having to re-authenticate if they move to a different page. The `history` object is used to have the user navigate without refreshing the page, something that would delete the users' authentication code from the Context of `Auth0Provider`:

```js
// src/index.js
import React from 'react';
import ReactDOM from 'react-dom';
import ApolloClient from 'apollo-boost';
import { ApolloProvider } from '@apollo/react-hooks';
import { BrowserRouter as Router } from 'react-router-dom';
import { createBrowserHistory } from 'history';
import App from './App';
import { Auth0Provider } from './react-auth0-spa';
import * as serviceWorker from './serviceWorker';

const history = createBrowserHistory();

const client = new ApolloClient({
  uri: 'http://localhost:4000/graphql'
});

const onRedirectCallback = () => {
  history.push(window.location.pathname);
};

ReactDOM.render(
  <Router>
    <Auth0Provider
      domain={process.env.REACT_APP_AUTH0_DOMAIN}
      client_id={process.env.REACT_APP_CLIENT_ID}
      redirect_uri={window.location.origin}
      onRedirectCallback={onRedirectCallback}
      audience={process.env.REACT_APP_API_IDENTIFIER}
    >
      <ApolloProvider client={client}>
        <App />
      </ApolloProvider>
    </Auth0Provider>
  </Router>,
  document.getElementById('root')
);

...

```

Any component that is nested within `Auth0Provider` is now able to send requests to the Auth0 service to authenticate the user by using the `useAuth0` Hook. This Hook returns multiple functions to help you with this, starting with the `loginWithRedirect` method that initiates the authentication with Auth0 and redirects the user to the login page. Also, the function `logout` is used by unauthenticated users and the const `isAuthenticated` shows the authentication status of a user. This functionality should be added in the file `src/App.js`:

```js
// src/App.js
import React from 'react';
import { Switch, Route } from 'react-router-dom';
import Events from './Events';
import { useAuth0 } from './react-auth0-spa';

function App() {
  const { loginWithRedirect, logout, isAuthenticated } = useAuth0();

  return (
    <div style={{ fontFamily: 'Helvetica' }}>
      <header
        style={{
          display: 'inline-block',
          width: '100%',
          backgroundColor: 'lightBlue',
          padding: '10 20px',
          textAlign: 'center',
          borderRadius: '5px',
        }}
      >
        <h1 style={{ color: 'white' }}>Events</h1>
        <button onClick={!isAuthenticated ? loginWithRedirect : logout}>
          {!isAuthenticated ? 'Login' : 'Logout'}
        </button>
      </header>

      ...

```

By clicking the _Login_ button, the Auth0 login screen gets opened. After logging in or creating an account, you get redirected to the page `http://localhost:3000`. If the authentication was successful, the _Login_ button has now changed into a _Logout_ button. Clicking this button will delete the authentication details of the user from the browsers.

> Whenever you make a change to the code of this project, the Create React App development server can restart and make the browser refresh the page. If this happens, you need to re-authenticate with Auth0 by clicking the _Login_ button again.

After completing the steps in this part of the section, you're able to authenticate with Auth0, meaning you can also start sending authenticated requests to the GraphQL server. You'll explore this in the next part of this section.

### Sending authenticated requests to the GraphQL server

Now that you're able to authenticate, it becomes possible to send authenticated requests to the server. For example, say you want to query an individual event. To do this, let's create a component to display a single event first. Create a file called `Event.js` in the `src` directory by running:

```bash
touch src/Event.js
```

And add this code block to that file:

```js
// src/Event.js
import React from 'react';
import { useParams } from 'react-router-dom';
import { gql } from 'apollo-boost';
import { useQuery } from '@apollo/react-hooks';

const GET_EVENT = gql`
  query getEvent($id: Int!) {
    event(id: $id) {
      id
      title
      date
      attendants {
        id
        name
      }
    }
  }
`;

function Events() {
  const { id } = useParams();

  const { loading, data, error } = useQuery(GET_EVENT, {
    variables: { id: parseInt(id) },
  });

  if (loading) return 'Loading...';
  if (error) return 'Something went wrong...';

  const { title, date, description, attendants, canEdit } =
    (data && data.event) || {};

  return (
    <ul style={{ listStyle: 'none', width: '100%', padding: '0' }}>
      <li
        style={{
          backgroundColor: 'lightGrey',
          marginBottom: '10px',
          padding: '10px',
          borderRadius: '5px',
        }}
      >
        <h2>{title}</h2>
        <em>{date}</em>

        <p>{description}</p>

        {attendants && (
          <p>
            <strong>Attendants:</strong>

            <ul>
              {attendants.map(attendant => (
                <li key={attendant.id}>{attendant.name}</li>
              ))}
            </ul>
          </p>
        )}
      </li>
    </ul>
  );
}

export default Events;
```

This file uses the `GET_EVENT` query to retrieve a single event based on the `id` for the route, which you can test by going to [`http://localhost:3000/event/2`](http://localhost:3000/event/2). The `useParams` Hook from `react-router-dom` gets the value for `id` from the route and the `useQuery` Hook retrieves the event. You can see there are no attendants displayed yet, as the information on the field `attendants` is only visible when you pass a valid JWT with the query.

Passing along a JWT requires you to set more parameters to the `useQuery` Hook, but first, let's make the single event route reachable from the `Events` component. You can use the `Link` component from `react-router-dom` and add this to the file `src/Events.js`:

```js
// src/Events.js
import React from 'react';
import { gql } from 'apollo-boost';
import { useQuery } from '@apollo/react-hooks';
import { Link } from 'react-router-dom';

  ...

  return (
    <ul style={{ listStyle: 'none', width: '100%', padding: '0' }}>
      {data.events.map(({ id, title, date }) => (
        <Link key={id} to={`/event/${id}`}>
          <li
            style={{
              backgroundColor: 'lightGrey',
              marginBottom: '10px',
              padding: '10px',
              borderRadius: '5px'
            }}
          >
            <h2>{title}</h2>
            <span style={{ fontStyle: 'italic' }}>{date}</span>
          </li>
        </Link>
      ))}
    </ul>
  );
}

export default Events;
```

In the file `src/Event.js`, the `useQuery` Hook must be altered so it can take a `context` object containing the header information as a parameter. This header information should include the field `authorization` that contains the JWT, which can be retrieved with the `getTokenSilently` method from the `useAuth0` Hook. As this is an asynchronous function, you need to call it from a React `useEffect` Hook and store the value in the local state using `useState`. You can do this by making the following changes to `src/Event.js`:

```js
// src/Event.js
import React from 'react';
import { useParams } from 'react-router-dom';
import { gql } from 'apollo-boost';
import { useQuery } from '@apollo/react-hooks';
import { useAuth0 } from './react-auth0-spa';

...

function Events() {
const { id } = useParams();
    const { isAuthenticated, getTokenSilently } = useAuth0();

  const [bearerToken, setBearerToken] = React.useState('');
  React.useEffect(() => {
    const getToken = async () => {
      const token = isAuthenticated ? await getTokenSilently() : '';

      setBearerToken(`Bearer ${token}`);
    };
    getToken();
  }, [getTokenSilently, isAuthenticated]);

  const { loading, data, error } = useQuery(GET_EVENT, {
    variables: { id: parseInt(id), bearerToken },
    context: {
      headers: {
        authorization: bearerToken
      }
    }
  });

  ...

```

After adding the asynchronous call to the `getTokenSilently` function, the token information for this user becomes available. When you're authenticated and visit a page with a single event, the token from Auth0 will be sent along in the document containing the query to retrieve this event. If you're token is valid, the response of the GraphQL server will also include a list with all the attendants of the event.

> **Note** If you don't see a list of attendants of an event, make sure that the values for `AUTH0_DOMAIN` and `API_IDENTIFIER` are the same for both the GraphQL server and the application that you're building in this post.

Next to showing events you also want the user's to be able to modify events, for which you need to send a document with a mutation instead of a query to the GraphQL server together with a valid JWT. Sending documents with mutations to the GraphQL server is very similar to how you implemented this for queries in the first section of this post. Instead of the `useQuery` Hook, you'll use the Hook called `useMutation`, which helps you with sending the document containing a mutation that allows you to edit events. In the previous post that explained how to create the GraphQL server, you added a mutation to edit both the `title` and `description` of an event. This mutation should also be used to edit the event from the React application.

But before creating the logic to mutate the data of an event, a `Form` component must be created. Using this component that you should create in the file `src/Form.js`, the event can be edited by adding the following code block to this file:

```js
// src/Form.js
import React from 'react';

const Form = ({ id, onSubmit, refetch, ...props }) => {
  const [title, setTitle] = React.useState(props.title);
  const [description, setDescription] = React.useState(props.description);

  const handleOnSubmit = e => {
    e.preventDefault();

    onSubmit({ variables: { id: parseInt(id), title, description } });
    refetch();
  };

  return (
    <div
      style={{
        backgroundColor: 'lightGrey',
        margin: '15px 0px',
        padding: '10px',
        borderRadius: '5px',
      }}
    >
      <h3>Edit event</h3>
      <form onSubmit={handleOnSubmit}>
        <p>
          <label for='title' style={{ marginRight: '10px' }}>
            Title:
          </label>
          <input
            id='title'
            type='text'
            value={title}
            onChange={e => setTitle(e.target.value)}
          />
        </p>

        <p>
          <label for='description' style={{ marginRight: '10px' }}>
            Description:
          </label>
          <input
            id='description'
            type='text'
            value={description}
            onChange={e => setDescription(e.target.value)}
          />
        </p>

        <button>Submit</button>
      </form>
    </div>
  );
};

export default Form;
```

This form has controlled input components, that use the local state to store changes when you type in the `input` elements. When you submit the form the `handleOnSubmit` function will be called, which calls both the `onSubmit` and `refetch` functions that were passed as props from the `Event` component. The `onSubmit` function will send the document with the mutation to the GraphQL server, while the `refect` function sends the document with the query to retrieve the event to the server. In the `Event` component you must import the `Form` component and add the logic to send the document with the mutation, by making these changes to `src/Event.js`:

```js
// src/Event.js
import React from 'react';
import { useParams } from 'react-router-dom';
import { gql } from 'apollo-boost';
import { useQuery, useMutation } from '@apollo/react-hooks';
import { useAuth0 } from './react-auth0-spa';
import Form from './Form';

...

const EDIT_EVENT = gql`
  mutation editEvent($id: Int!, $title: String!, $description: String!) {
    editEvent(id: $id, title: $title, description: $description) {
      title
      description
    }
  }
`;

function Event() {

  ...

  const [editEvent] = useMutation(EDIT_EVENT, {
    context: {
      headers: {
        authorization: bearerToken,
      },
    },
  });

  if (loading) return 'Loading...';
  if (error) return 'Something went wrong...';

  return (
    <>

      ...

      {isAuthenticated && (
        <Form
          id={id}
          onSubmit={editEvent}
          refetch={refetch}
          title={title}
          description={description}
        />
      )}
    </>
  );
}

export default Event;

```

After making these changes, you can start editing the event when you're logged in as the `Form` component only gets displayed when the user is authenticated. However, you also want to implement authorization as you don't want every authenticated user to be able to change the information of an event. How to do this is explained in the next part of this section.

### Handle Authorization

Next to authentication, another important part of your application is authorization as you want to distinguish between "regular" users and users that can modify your events. For this, you can use permissions and user roles, which are part of the Auth0 authentication service and can be added to the GraphQL server that you're using together with the React application that you're building in this post.

To handle authorization you first need to perform several actions in the Auth0 Dashboard:

- Add [permissions](https://auth0.com/docs/dashboard/guides/apis/add-permissions-apis) to the API that you've created in the GraphQL Server tutorial, which you can do by going to the [API](https://manage.auth0.com/#/apis/) page in the Auth0 dashboard. On this page, you need to create a new `permission` called `edit:events` on the _Permissions_ tab.
- On this same page, you need to make sure that both checkboxes for _RBAC Settings_ on the _Settings_ tab are checked, the first one enables Role-Based Access Control (RBAC) for the GraphQL server and the second checkbox adds the permissions for the user to the JWT.
- You need to add this permission to an [user role](https://auth0.com/docs/dashboard/guides/roles/add-permissions-roles) and add a user to this role. Adding a new user role can also be done by visiting the [Roles](https://manage.auth0.com/#/roles/) page. Here you must create a new role called `admin` using the _Create role_ button, after creating the role you must add the `edit:events` permission and a user to it.
- Adding a `permission` and a user to this role can be done by clicking on the newly created role on the [Roles](https://manage.auth0.com/#/roles/) page. On the page that opens you can add the `edit:events` permission to this role from the _Permissions_ tab and the user from the _Users_ tab, which must be the user you're using to log in to the application you've created in this post.

These changes were needed to make it possible to add the permission to edit events to users and add these permissions to the JWT that's created for that user. The validation of the JWT is done on the GraphQL server, meaning that the logic to check if a user has the correct permission to edit an event must be added to the GraphQL server as well. In the code for the GraphQL server you must find the file `src/index.js` and make the following changes:

```js
// src/index.js

...

// Provide resolver functions for your schema fields
const resolvers = {

  ...

  event: async ({ id }, context) => {
    const { db, token } = await context();

    const { error, decoded } = await isTokenValid(token);

    const event = await db.collection('events').findOne({ id });

    const canEdit = decoded
      ? decoded.permissions && decoded.permissions.includes('edit:events')
      : false;

    return { ...event, attendants: !error ? event.attendants : null, canEdit };
  },

  ...

```

The `isTokenValid` function decoded the JWT and inside the decoded JWT the field `permissions` can be found. This field contains an array with permissions for this user and if the permission `edit:events` that you've created before is present. If the user's JWT has this permission it means that the user can edit the event and the value for `canEdit` that is returned by the `event` query is true. Now, this query returns the field `canEdit` you must also add this field to the schema for this query in `src/schema.js`:

```js
// src/schema.js

...

// Construct a schema, using GraphQL schema language
const schema = buildSchema(`
    type Event {
        id: ID!
        title: String!
        description: String
        date: String
        attendants: [Person!]
        canEdit: Boolean
    }

    ...

```

> **Note** The updated code for the GraphQL server can also be found in [this](https://github.com/auth0-blog/auth0-graphql-server) repository on Github, where you've got to check out the `permissions` branch.

In the React application, you must update the query that's used to retrieve a single event, by adding the field `canEdit` here as well. When the field `canEdit` is true, the user is both authenticated and has the correct permissions to edit events. Therefore you can make changes to `src/Event.js` to not only ask for the field `canEdit`, but also use this value to determine whether or not the `Form` component must be displayed:

```js
// src/Event.js

...

const GET_EVENT = gql`
  query getEvent($id: Int!) {
    event(id: $id) {
      id
      title
      date
      description
      attendants {
        id
        name
      }
      canEdit
    }
  }
`;

...

function Event() {

  ...

  const { title, date, description, attendants, canEdit } =
    (data && data.event) || {};

  return (
    <>

      ...

      {canEdit && (
        <Form
          id={id}
          onSubmit={editEvent}
          refetch={refetch}
          title={title}
          description={description}
        />
      )}
    </>
  );
}

export default Event;
```

If a user is authenticated but doesn't have the correct permission to edit events, the `Form` component won't be displayed making it unable for the user to edit the event details.

## Conclusion

In this tutorial, you've created a React application, that uses a GraphQL server to retrieve its data and Auth0 for authorization and authentication. You've learned how to set up a basic React application with routing, and Apollo to fetch data from a React server using queries and mutations. By using Auth0, authentication and authorization are added to the application, giving users different permissions for taking actions. The full code for this tutorial can be found on Github, as well as for the tutorial about creating a GraphQL server.
