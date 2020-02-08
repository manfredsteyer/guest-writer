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

**TL;DR:** This post will show how you can query, mutate, and consume data from a GraphQL server using a React client application, which implements authentication using Auth0 to guard access to privileged data.

Web applications have rapidly increased in complexity as more and more technologies and tools for the web are available nowadays. One of these technologies is React, a JavaScript library for creating Single-Page Applications (SPA) or mobile applications.

Another technology that has arisen the past few years is GraphQL, which simplifies how you can query and mutate data over HTTP. When used together, these technologies let you create future-proof web applications that can be secured using Auth0.

In this post, you'll learn how to develop a secure web application with **React and GraphQL using Apollo**. Not only will this application handle authentication, logged in users can have different authorization levels that define the actions they can take in the application.

## What Will You Build

The React application that you'll build in this tutorial will display a list of events and allow you to manage the event attendees. The user interface will consist of two pages: a page to list the events and a page to edit the events. The application will consume the event data from a GraphQL server that is set up to handle authentication and authorization with Auth0.

## Prerequisites

You'll need to have `node` and `npm` installed on your machine. If you don't have these installed yet, you can find the installation instructions [here](https://nodejs.org/en/download/).

You'll also need a live GraphQL server to better showcase the integration of GraphQL with React. But don't worry, we got you covered. 

For the first part of the tutorial, you can use this [demo GraphQL server](https://auth0-graphql-simple.herokuapp.com/playground) that we are hosting on Heroku, which is a basic server that lets you read data:

```bash
https://auth0-graphql-simple.herokuapp.com/playground
```

For the second part of the tutorial, you'll need a more complex server that supports reading and writing data and that implements authorization to restrict access to resources. For that, you'll use the [GraphQL server from the first chapter of this tutorial](https://github.com/auth0-blog/auth0-graphql-server).

To run that server, you need to follow the steps in the _Getting started_ section of the repo's `README` file, including [adding your Auth0 information](https://auth0.com/blog/build-and-secure-a-graphql-server-with-node-js/#Securing-a-GraphQL-Server-with-Auth0). If you don't have an Auth0 account yet, you can <a href="https://auth0.com/signup" data-amp-replace="CLIENT_ID" data-amp-addparams="anonId=CLIENT_ID(cid-scope-cookie-fallback-name)">register one for free here</a>. Also, I'm assuming you already have prior knowledge about JavaScript and React and know how you can create components and update state.

> **Note** You will need to write down the values for `AUTH0_DOMAIN` and `API_IDENTIFIER` from the `.env` file of the GraphQL server, as you'll also need these values for this post.

## What is GraphQL?

In short, GraphQL is a query language for APIs that lets you query and mutate data over HTTP. It was publicly released by Facebook in 2015 to help them provide their mobile applications with just the data they need instead of sending "raw" data over REST that needs to be filtered in the frontend. If you feel like you need to learn more about GraphQL, you can read a longer description in the [first part](https://github.com/auth0-blog/auth0-graphql-server) of this series about GraphQL.

## Creating a SPA with React

To quickly create a React application, you can use [`create-react-app`](https://github.com/facebook/create-react-app), which is an integrated toolchain that provides you with a great developer experience. `create-react-app` creates a frontend build pipeline that you can pair any backend you need to consume, such as a GraphQL API.  

### Getting started with React

Instead of installing `create-react-app` globally in your machine, you can use `npx` to execute the remote package locally to scaffold your React application. `npx` is a package runner tool that comes with installed with  `npm v5.2+`.

Run the following command in your terminal to create a new React project:

```bash
npx create-react-app auth-graphql-app
```

Once the project is created, you'll see a new directory called `auth-graphql-app` under the directory from where you executed the command. This directory holds all the boilerplate code of your React application.

You can move into this directory and start the application by running the following command:

```
cd auth-graphql-app && npm start
```

The command `npm start` will execute a start script defined in `package.json`, which compiles and runs your code on a local development server. You can access your live React application by opening [http://localhost:3000](http://localhost:3000) in your browser.

One of the most important files in your project is `src/index.js`, which is the entry point of your React application. In this file, a component called `App` is rendered by `ReactDOM`. All the components that you'd like to render in the browser must be children of the `App` component, which is defined in the `src/App.js` file.
 
You can think of the `App` component as the top-level component of your application. As such, `App` is the best place to bootstrap the routing of your application. React doesn't provide a routing module out-of-the-box; however, you can use a popular and well-maintained package called `react-router-dom` to take care of your application routing needs. 
 
Install that routing package from npm by running the command below in your terminal:

```
npm install react-router-dom history
```

With `react-router-dom`, you can import a `Router` component that will be used by your application to change between pages dynamically without triggering a full-page refresh. The `Router` component must be added at the top level in `src/index.js` and it must wrap the `App` component.

Update `src/index.js` as follows: 

```jsx harmony
// src/index.js

import React from "react";
import ReactDOM from "react-dom";
import { BrowserRouter as Router } from "react-router-dom";
import App from "./App";

ReactDOM.render(
  <Router>
    <App />
  </Router>,
  document.getElementById("root")
);
```

In turn, the routes for your application can be defined within the `App` component. Replace the code present in the `src/App.js` file with the following:

```jsx harmony
// src/App.js

import React from "react";
import { Switch, Route } from "react-router-dom";

function App() {
  return (
    <Switch>
      <Route path="/event/:id">Event</Route>
      <Route path="*">All Events</Route>
    </Switch>
  );
}

export default App;
```

With the above changes you've created two routes for your application: `/event/:id` and `*`. The `Switch` component first tries to find the route for a single event identified by an `:id` parameter, and, if it's not found, it directs to the _All Events_ route. The `*` value of the `path` attribute marks that route as a default route and it has to be the last in the route hierarchy.

The route for a single event can be found by visiting a page with the url `http://localhost:3000/event` followed by the event `id`, e.g: [`http://localhost:3000/event/123`](http://localhost:3000/event/123). Later on in this tutorial, you'll add the data to these routes.

### Cleaning up the project

You can delete the following files from the project as you won't need them:

```
src
|-- App.css
|-- App.test.js
|-- index.css
|-- logo.svg
|-- serviceWorker.js
|-- setupTests.js
```

You can delete all these files easily by running this command:

```bash
rm src/App.css src/App.test.js src/index.css src/logo.svg src/serviceWorker.js src/setupTests.js 
```

The project is now ready to be connected to a GraphQL server.

## Handling Environmental Variables

Your application may use configuration variables that may change depending on the environment where it is running. As such, instead of hard coding that data in your application components, it's better to store them in a centralized location like a local environment file.
 
When you create an application using Create React App, you can create a `.env` file under the root project directory to define environmental variables. You can access the variables within your application components by using the `process.env` object.

The variables defined within the `.env` file of a Create React App project must be prefixed with `REACT_APP_`; otherwise, you won't be able to access them as [any other variables except `NODE_ENV` will be ignored](https://create-react-app.dev/docs/adding-custom-environment-variables/) to avoid accidentally exposing a private key on the machine that could have the same name.

From the root directory of the project, run this command to create the `.env` file:

```bash
touch .env
```

Place the following code in this file:

```
REACT_APP_APOLLO_CLIENT_URI=https://auth0-graphql-simple.herokuapp.com/graphql
```

The first configuration variable that you are defining is `REACT_APP_APOLLO_CLIENT_URI`, which represents the GraphQL endpoint that you want to connect your React app with. You'll be using this variable in the next section.

Keep in mind that whenever you change any environment variables you need to restart the development server for the application to consume the changes made to `.env`.

## Using GraphQL with Apollo

**EDITOR'S NOTE:** Let's use this section to introduce Apollo and any of its benefit in relation with GraphQL and React. Since the readers are using the remote demo server at this point, they don't need to worry about setting up the local server yet.


### Set up Apollo client with React

**EDITOR'S NOTE:** Let's tell the readers here that we are going to teach them first how to query/read data using that remote demo server and that we'll use a more complex server later on to learn about writing data to secured endpoints.

Before you can send documents to the GraphQL server, you need to set up the connection with the server, which can be done by adding the package `apollo-boost`. This package configures a GraphQL client for your application with Apollo's recommended settings for caching, state management, and error handling. Next to `apollo-boost`, you also need to install the packages `@apollo/react-hooks` and `graphql`. With `@apollo/react-hooks`, you can handle queries and mutations from your React components, while `graphql` is needed to use GraphQL's query language inside your application. To add these packages to your application, run the following command from your terminal:

```
npm install apollo-boost @apollo/react-hooks graphql
```

In the file `src/index.js`, you can subsequently import the method `ApolloClient` from [`apollo-boost`](https://github.com/apollographql/apollo-client/tree/master/packages/apollo-boost) that will create the GraphQL client for you. This method needs the endpoint to the GraphQL server that is running on `http://localhost:4000/` to get you started. By default, the client expects the GraphQL server to be running on the `/graphql` endpoint. Creating the client is done by making the changes below to `src/index.js`:

```jsx harmony
// src/index.js

import React from "react";
import ReactDOM from "react-dom";
import { BrowserRouter as Router } from "react-router-dom";
import ApolloClient from "apollo-boost";
import App from "./App";

const client = new ApolloClient({
	uri: process.env.REACT_APP_APOLLO_CLIENT_URI
});

ReactDOM.render(
  <Router>
    <App />
  </Router>,
  document.getElementById("root")
);
```

But creating a client isn't sufficient to connect your React application to the GraphQL server, as you also need to create a Provider that wraps your application and makes it possible to access the client from everywhere in your project. This Provider uses the Context API from React. I [previously wrote a blog post](https://auth0.com/blog/handling-authentication-in-react-with-context-and-hooks/) about it, so you can check that out if you aren't familiar with it.

You can create a Provider by importing `ApolloProvider` from `@apollo/react-hooks` in the file `src/index.js` and pass the client to it. The `ApolloProvider` must be placed inside the `Router` to wrap the `App` component that is rendered by `ReactDOM` by making the following changes:

```jsx harmony
// src/index.js

import React from "react";
import ReactDOM from "react-dom";
import { BrowserRouter as Router } from "react-router-dom";
import ApolloClient from "apollo-boost";
import { ApolloProvider } from "@apollo/react-hooks";
import App from "./App";

const client = new ApolloClient({
	uri: process.env.REACT_APP_APOLLO_CLIENT_URI
});

ReactDOM.render(
  <Router>
    <ApolloProvider client={client}>
      <App />
    </ApolloProvider>
  </Router>,
  document.getElementById("root")
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

```jsx harmony
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

```jsx harmony
// src/App.js

import React from "react";
import { Switch, Route } from "react-router-dom";
import Events from "./Events";

function App() {
  const appStyle = { fontFamily: "Helvetica" };
  const headerStyle = {
    display: "inline-block",
    width: "100%",
    backgroundColor: "lightBlue",
    padding: "10 20px",
    textAlign: "center",
    borderRadius: "5px"
  };
  const h1Style = { color: "white" };

  return (
    <div style={appStyle}>
      <header style={headerStyle}>
        <h1 style={h1Style}>Events</h1>
      </header>
      <Switch>
        <Route path="/event/:id">Event</Route>
        <Route path="*">
          <Events />
        </Route>
      </Switch>
    </div>
  );
}

export default App;
```

If you open the browser again, you can see that the list of events is being displayed together with the `title` and `date` of each event. This page could also use some styling, which can be done by changing the following in the file `src/Events.js`:

```jsx harmony
// src/Events.js

// ...

function Events() {
  const { loading, data, error } = useQuery(GET_EVENTS);

  const ulStyle = { listStyle: "none", width: "100%", padding: "0" };
  const liStyle = {
    backgroundColor: "lightGrey",
    marginBottom: "10px",
    padding: "10px",
    borderRadius: "5px"
  };
  const spanStyle = { fontStyle: "italic" };

  if (loading) return "Loading...";
  if (error) return "Something went wrong...";

  return (
    <ul style={ulStyle}>
      {data.events.map(({ id, title, date }) => (
        <li key={id} style={liStyle}>
          <h2>{title}</h2>
          <span style={spanStyle}>{date}</span>
        </li>
      ))}
    </ul>
  );
}

export default Events;
```

But having just a list of all the events is not sufficient, as you also want to display the attendants of that event. However, this information is private and requires you to send a valid JWT along with your query. This JWT can be retrieved by sending a request to the Auth0 authentication service, which you'll do in the next section of this post.

## Securing a React app

You'll use Auth0 to secure your React application and your GraphQL server easily. By sending a request to the Auth0 authentication service with your credentials, you'll retrieve a JWT that can be validated by a GraphQL server.

For this section, you'll start using a more complex server that exposes an API with protected endpoints to write data, along the a few read endpoints to get data to display on your React application. 

If you followed the previous tutorial, [_Build and Secure a GraphQL Server with Node.js_](https://auth0.com/blog/build-and-secure-a-graphql-server-with-node-js/), you can use the server you built there. Otherwise, I'll give you instructions on how to clone the code from that tutorial to set up a local GraphQL server. Let's start first by setting up your React application to talk to a secured server.

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

```jsx harmony
// src/react-auth0-spa.js

import React, { useState, useEffect, useContext } from "react";
import createAuth0Client from "@auth0/auth0-spa-js";

const DEFAULT_REDIRECT_CALLBACK = () =>
  window.history.replaceState({}, document.title, window.location.pathname);

export const Auth0Context = React.createContext();

export const useAuth0 = () => useContext(Auth0Context);

export const Auth0Provider = ({
  children,
  onRedirectCallback = DEFAULT_REDIRECT_CALLBACK,
  ...initOptions
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

      if (window.location.search.includes("code=")) {
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

  const contextValue = {
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
    logout: (...p) => auth0Client.logout(...p)
  };

  return (
    <Auth0Context.Provider value={contextValue}>
      {children}
    </Auth0Context.Provider>
  );
};
```

This file sets up the connection with Auth0 and returns a Provider called `Auth0Provider` and a Hook. This Provider is similar to `ApolloProvider` and needs your Auth0 credentials, making it possible to use the `useAuth0` Hook to connect with Auth0 from any components that are nested inside.

Before adding the `Auth0Provider` to your project you need to register your React application with Auth0. If you have not created an Auth0 account yet, I invite to <a href="https://auth0.com/signup" data-amp-replace="CLIENT_ID" data-amp-addparams="anonId=CLIENT_ID(cid-scope-cookie-fallback-name)">sign up for a free Auth0 account here</a>.


Once you log into your Auth0 account, open the [Applications section of the Auth0 dashboard](https://manage.auth0.com/#/applications) and click on the _Create Application_ button. 

In the dialog box that comes up, you need to set the name for your application, for example _EventsQL_, and select _Single Web Page Application_ as the application type. Once that's done, click on the _Create_ button.

On the new page that loads, click on the _Settings_ tab to find the values for `Domain` and `Client ID` from Auth0 that you'll need to set up the `Auth0Provider` of your React application.

The values for `Domain` and `Client ID` must be stored  in your `.env` file as follows:

```
REACT_APP_APOLLO_CLIENT_URI=https://auth0-graphql-simple.herokuapp.com/graphql
REACT_APP_AUTH0_DOMAIN=YOUR_AUTH0_DOMAIN
REACT_APP_CLIENT_ID=YOUR_CLIENT_ID
```

The values of `YOUR_AUTH0_DOMAIN` and `YOUR_CLIENT_ID` must be replaced by the values for `Domain` and `Client ID` from the _Settings_ tab of the Auth0 application page.

On that same tab, scroll down to find the _Allowed Callback URLs_, _Allowed Logout URLs_ and _Allowed Web Origins_ fields. Set each of these values to the address where your React application is running, which in this case is `http://localhost:3000`. Once that's done, scroll down and click on the _Save Changes_ button.

You can now restart the development server of Create React App by running `npm start` again. Your Auth0 credentials should now be available to use in the application to set up `Auth0Provider` in the file `src/index.js` and be placed around the `ApolloProvider` component. Also, `Auth0Provider` should call a callback function to redirect the user to the correct page after authentication. This function is used to clear the user's authentication code from the url to prevent them from having to re-authenticate if they move to a different page. The `history` object is used to have the user navigate without refreshing the page, something that would delete the users' authentication code from the Context of `Auth0Provider`. This value is created with the `createBrowserHistory` function from the `history` package, which is a major dependency of `react-router-dom` and therefore doesn't have to be installed separately:

```jsx harmony
// src/index.js

import React from "react";
import ReactDOM from "react-dom";
import ApolloClient from "apollo-boost";
import { ApolloProvider } from "@apollo/react-hooks";
import { BrowserRouter as Router } from "react-router-dom";
import { createBrowserHistory } from "history";
import App from "./App";
import { Auth0Provider } from "./react-auth0-spa";

const history = createBrowserHistory();

const client = new ApolloClient({
	uri: process.env.REACT_APP_APOLLO_CLIENT_URI
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
  document.getElementById("root")
);
```

Any component that is nested within `Auth0Provider` is now able to send requests to the Auth0 service to authenticate the user by using the `useAuth0` Hook. This Hook returns multiple functions to help you with this, starting with the `loginWithRedirect` method that initiates the authentication with Auth0 and redirects the user to the login page. Also, the function `logout` is used by unauthenticated users, and the const `isAuthenticated` shows the authentication status of a user. This functionality should be added in the file `src/App.js`:

```jsx harmony
// src/App.js

import React from "react";
import { Switch, Route } from "react-router-dom";
import Events from "./Events";
import { useAuth0 } from "./react-auth0-spa";

function App() {
  const appStyle = { fontFamily: "Helvetica" };
  const headerStyle = {
    display: "inline-block",
    width: "100%",
    backgroundColor: "lightBlue",
    padding: "10 20px",
    textAlign: "center",
    borderRadius: "5px"
  };
  const h1Style = { color: "white" };

  const { loginWithRedirect, logout, isAuthenticated } = useAuth0();

  return (
    <div style={appStyle}>
      <header style={headerStyle}>
        <h1 style={h1Style}>Events</h1>
        <button onClick={!isAuthenticated ? loginWithRedirect : logout}>
          {!isAuthenticated ? "Login" : "Logout"}
        </button>
      </header>
      <Switch>
        <Route path="/event/:id">Event</Route>
        <Route path="*">
          <Events />
        </Route>
      </Switch>
    </div>
  );
}

export default App;
```

By clicking the _Login_ button, the Auth0 login screen gets opened. After logging in or creating an account, you be be asked to authorize your Auth0 application, "EventsQL", to access your Auth0 tenant. The authorization request will also show you the permissions that the Auth0 application is requesting, such as access to your profile and email information.

![Auth0 application requesting authorization to access profile information](https://cdn.auth0.com/blog/developing-secure-web-applications-with-react-and-graphql/authentication-content.png)

Once you grant access or consent, you get redirected to [`http://localhost:3000`](http://localhost:3000/). If the authentication was successful, the _Login_ button has now changed into a _Logout_ button. Clicking this button will delete the authentication details of the user from the browsers.

> Whenever you make a change to the code of this project, the Create React App development server can restart and make the browser refresh the page. If this happens, you need to re-authenticate with Auth0 by clicking the _Login_ button again.

After completing the steps in this part of the section, you're able to authenticate with Auth0, meaning you can also start sending authenticated requests to the GraphQL server. You'll explore this in the next sections.

## Setting Up a GraphQL Server

In this section, you'll use a more complex GraphQL server that supports both read and write operations and that implements an authorization layer to restrict access to resources. This server is the result of following the [_Build and Secure a GraphQL Server with Node.js_](https://auth0.com/blog/build-and-secure-a-graphql-server-with-node-js/) tutorial. If you've completed that tutorial, you can proceed to the next subsection, _Sending authenticated requests to the GraphQL server_.

Otherwise, follow these instructions to set up the server using the code from that tutorial, which is hosted in a [GitHub repository](https://github.com/auth0-blog/auth0-graphql-server):

- Anywhere in your system, run the following command to clone the repo:

```bash
git clone git@github.com:auth0-blog/auth0-graphql-server.git
```

- Move into the directory created for the project:

```bash
cd auth0-graphql-server
```

- Follow the steps on the [Securing a GraphQL Server with Auth0](https://auth0.com/blog/build-and-secure-a-graphql-server-with-node-js/#Securing-a-GraphQL-Server-with-Auth0) section of the previous chapter to set up an Auth0 API for your server, which is an API that you define within your Auth0 tenant and that you can consume from your applications to process authentication and authorization requests.

- Create a new file called `.env` under the root project directory and add your Auth0 information to it like this:

```bash
AUTH0_DOMAIN=auth0blog.auth0.com
API_IDENTIFIER=https://graphql-api
```

The value of `AUTH0_DOMAIN` follows this pattern `<AUTH0-TENANT-NAME>.auth0.com`, where `AUTH0-TENANT-NAME` is the name of the Auth0 tenant where your Auth0 API is being hosted. This is exactly the same **Domain** value that you can find under the _Settings_ tab of your Auth0 application, _EventQL_. 

The value of `API_IDENTIFIER` is the Identifier value you created for the API. 

After creating this file and adding the correct values to it, you can install the project dependencies and start the GraphQL server by running this:

```bash
npm install && npm start
```

The GraphQL server will become available at [`http://localhost:4000/graphql`](http://localhost:4000/graphql). Head back to your React application and open `.env`. Replace the value of `REACT_APP_APOLLO_CLIENT_URI` with `http://localhost:4000/graphql`.

Also, you need the add a new `REACT_APP_API_IDENTIFIER` variable to the React `.env` file. The value of `API_IDENTIFIER` must be the same value of the `API_IDENTIFIER` you created earlier to set up the Auth0 API.

Your final React `.env` file will look similar to this:

```bash
REACT_APP_APOLLO_CLIENT_URI=http://localhost:4000/graphql
REACT_APP_AUTH0_DOMAIN=YOUR_AUTH0_DOMAIN
REACT_APP_CLIENT_ID=YOUR_CLIENT_ID
REACT_APP_API_IDENTIFIER=GRAPHQL_SERVER_API_IDENTIFIER
```

To test that the Graph server is up and running, you can use the interactive playground at [`http://localhost:4000/playground`](http://localhost:4000/playground). Using this GraphQL Playground interface, you can inspect the schema of this server or send documents containing queries or mutations to it. An example of a document with a query that can be handled by this GraphQL server is:

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

The response of this query will be the full list events, including its `title`, `date`, and the `name` of every attendant of the event. This response will be in JSON and looks like the following:

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

## Making Secured Requests to the GraphQL Server

Now that you're able to authenticate, it becomes possible to send authenticated requests to the server. For example, say you want to query an individual event. To do this, let's create a component to display a single event first. Create a file called `Event.js` in the `src` directory by running:

```bash
touch src/Event.js
```

And add this code block to that file:

```jsx harmony
// src/Event.js

import React from "react";
import { useParams } from "react-router-dom";
import { gql } from "apollo-boost";
import { useQuery } from "@apollo/react-hooks";

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

function Event() {
  const { id } = useParams();

  const { loading, data, error } = useQuery(GET_EVENT, {
    variables: { id: parseInt(id) }
  });

  if (loading) return "Loading...";
  if (error) return "Something went wrong...";

  const { title, date, description, attendants, canEdit } =
    (data && data.event) || {};

  const ulStyle = {
    listStyle: "none",
    width: "100%",
    backgroundColor: "lightGrey",
    margin: "10px 0",
    padding: "10px",
    borderRadius: "5px"
  };

  return (
    <div style={ulStyle}>
      <h2>{title}</h2>
      <em>{date}</em>

      <p>{description}</p>

      {attendants && (
        <div>
          <strong>Attendants:</strong>

          <ul>
            {attendants.map(attendant => (
              <li key={attendant.id}>{attendant.name}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
}

export default Event;
```

This file uses the `GET_EVENT` query to retrieve a single event based on the `id` for the route, which you can test by going to [`http://localhost:3000/event/2`](http://localhost:3000/event/2). The `useParams` Hook from `react-router-dom` gets the value for `id` from the route, and the `useQuery` Hook retrieves the event. You can see there are no attendants displayed yet, as the information on the field `attendants` is only visible when you pass a valid JWT with the query.

Passing along a JWT requires you to set more parameters to the `useQuery` Hook, but first, let's make the single event route reachable from the `Events` component. You can use the `Link` component from `react-router-dom` and add this to the file `src/Events.js`:

```jsx harmony
// src/Events.js

import React from "react";
import { gql } from "apollo-boost";
import { useQuery } from "@apollo/react-hooks";
import { Link } from "react-router-dom";

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

  const ulStyle = { listStyle: "none", width: "100%", padding: "0" };
  const liStyle = {
    backgroundColor: "lightGrey",
    marginBottom: "10px",
    padding: "10px",
    borderRadius: "5px"
  };
  const spanStyle = { fontStyle: "italic" };

  if (loading) return "Loading...";
  if (error) return "Something went wrong...";

  return (
    <ul style={ulStyle}>
      {data.events.map(({ id, title, date }) => (
        <Link key={id} to={`/event/${id}`}>
          <li style={liStyle}>
            <h2>{title}</h2>
            <span style={spanStyle}>{date}</span>
          </li>
        </Link>
      ))}
    </ul>
  );
}

export default Events;
```

In the file `src/Event.js`, the `useQuery` Hook must be altered so it can take a `context` object containing the header information as a parameter. This header information should include the field `authorization` that contains the JWT, which can be retrieved with the `getTokenSilently` method from the `useAuth0` Hook. As this is an asynchronous function, you need to call it from a React `useEffect` Hook and store the value in the local state using `useState`. You can do this by making the following changes to `src/Event.js`:

```jsx harmony
// src/Event.js

import React from "react";
import { useParams } from "react-router-dom";
import { gql } from "apollo-boost";
import { useQuery } from "@apollo/react-hooks";
import { useAuth0 } from "./react-auth0-spa";

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

function Event() {
  const { id } = useParams();

  const { isAuthenticated, getTokenSilently } = useAuth0();
  const [bearerToken, setBearerToken] = React.useState("");

  React.useEffect(() => {
    const getToken = async () => {
      const token = isAuthenticated ? await getTokenSilently() : "";

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

  if (loading) return "Loading...";
  if (error) return "Something went wrong...";

  const { title, date, description, attendants, canEdit } =
  (data && data.event) || {};

  const ulStyle = {
    listStyle: "none",
    width: "100%",
    backgroundColor: "lightGrey",
    margin: "10px 0",
    padding: "10px",
    borderRadius: "5px"
  };

  return (
    <div style={ulStyle}>
      <h2>{title}</h2>
      <em>{date}</em>

      <p>{description}</p>

      {attendants && (
        <div>
          <strong>Attendants:</strong>

          <ul>
            {attendants.map(attendant => (
              <li key={attendant.id}>{attendant.name}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
}

export default Event;
```

After adding the asynchronous call to the `getTokenSilently` function, the token information for this user becomes available. When you're authenticated and visit a page with a single event, the token from Auth0 will be sent along in the document containing the query to retrieve this event. If you're token is valid, the response of the GraphQL server will also include a list with all the attendants of the event.

> **Note** If you don't see a list of attendants of an event, make sure that the values for `AUTH0_DOMAIN` and `API_IDENTIFIER` are the same for both the GraphQL server and the application that you're building in this post.

Next to showing events, you also want the user's to be able to modify events, for which you need to send a document with a mutation instead of a query to the GraphQL server together with a valid JWT. Sending documents with mutations to the GraphQL server is very similar to how you implemented this for queries in the first section of this post. Instead of the `useQuery` Hook, you'll use the Hook called `useMutation`, which helps you with sending the document containing a mutation that allows you to edit events. In the previous post that explained how to create the GraphQL server, you added a mutation to edit both the `title` and `description` of an event. This mutation should also be used to edit the event from the React application.

But before creating the logic to mutate the data of an event, a `Form` component should be created by running:

```bash
touch src/Form.js
```

This creates the file `src/Form.js`, to which the following code block must be added:

```jsx harmony
// src/Form.js

import React from "react";

const Form = ({ id, onSubmit, refetch, ...props }) => {
  const [title, setTitle] = React.useState(props.title);
  const [description, setDescription] = React.useState(props.description);

  const handleOnSubmit = e => {
    e.preventDefault();

    onSubmit({ variables: { id: parseInt(id), title, description } });
    refetch();
  };

  const containerStyle = {
    backgroundColor: "lightGrey",
    margin: "15px 0px",
    padding: "10px",
    borderRadius: "5px"
  };

  const labelStyle = { marginRight: "10px" };

  return (
    <div style={containerStyle}>
      <h3>Edit event</h3>
      <form onSubmit={handleOnSubmit}>
        <p>
          <label htmlFor="title" style={labelStyle}>
            Title:
          </label>
          <input
            id="title"
            type="text"
            value={title}
            onChange={e => setTitle(e.target.value)}
          />
        </p>

        <p>
          <label htmlFor="description" style={labelStyle}>
            Description:
          </label>
          <input
            id="description"
            type="text"
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

This form has controlled input components that use the local state to store changes when you type in the `input` elements. When you submit the form, the `handleOnSubmit` function will be called, which calls both the `onSubmit` and `refetch` functions that were passed as props from the `Event` component. The `onSubmit` function will send the document with the mutation to the GraphQL server, while the `refect` function sends the document with the query to retrieve the event to the server. 

In the `Event` component, you must import the `Form` component and add the logic to send the document with the mutation. Therefore you need to define the mutation to edit the event and use it as a parameter in the `useMutation` Hook. The `refetch` function must be destructured from the `useQuery` Hook and passed to the `Form` component, together with the `editEvent` function by making these changes to `src/Event.js`:

```js
// src/Event.js

import React from "react";
import { useParams } from "react-router-dom";
import { gql } from "apollo-boost";
import { useQuery, useMutation } from "@apollo/react-hooks";
import { useAuth0 } from "./react-auth0-spa";
import Form from "./Form";

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

const EDIT_EVENT = gql`
  mutation editEvent($id: Int!, $title: String!, $description: String!) {
    editEvent(id: $id, title: $title, description: $description) {
      title
      description
    }
  }
`;

function Event() {
  const { id } = useParams();

  const { isAuthenticated, getTokenSilently } = useAuth0();
  const [bearerToken, setBearerToken] = React.useState("");

  React.useEffect(() => {
    const getToken = async () => {
      const token = isAuthenticated ? await getTokenSilently() : "";

      setBearerToken(`Bearer ${token}`);
    };
    getToken();
  }, [getTokenSilently, isAuthenticated]);

  const { loading, data, error, refetch } = useQuery(GET_EVENT, {
    variables: { id: parseInt(id), bearerToken },
    context: {
      headers: {
        authorization: bearerToken
      }
    }
  });

  const [editEvent] = useMutation(EDIT_EVENT, {
    context: {
      headers: {
        authorization: bearerToken
      }
    }
  });

  if (loading) return "Loading...";
  if (error) return "Something went wrong...";

  const { title, date, description, attendants, canEdit } =
    (data && data.event) || {};

  const ulStyle = {
    listStyle: "none",
    width: "100%",
    backgroundColor: "lightGrey",
    margin: "10px 0",
    padding: "10px",
    borderRadius: "5px"
  };

  return (
    <div style={ulStyle}>
      <h2>{title}</h2>
      <em>{date}</em>

      <p>{description}</p>

      {attendants && (
        <div>
          <strong>Attendants:</strong>

          <ul>
            {attendants.map(attendant => (
              <li key={attendant.id}>{attendant.name}</li>
            ))}
          </ul>
        </div>
      )}

      {isAuthenticated && (
        <Form
          id={id}
          onSubmit={editEvent}
          refetch={refetch}
          title={title}
          description={description}
        />
      )}
    </div>
  );
}

export default Event;
```

After making these changes, you can start editing the event when you're logged in as the `Form` component only gets displayed when the user is authenticated. However, you also want to implement authorization as you don't want every authenticated user to be able to change the information of an event. How to do this is explained in the next part of this section.

## Handling Authorization

Next to authentication, another important part of your application is authorization as you want to distinguish between "regular" users and users that can modify your events. For this, you can use permissions and user roles, which are part of the Auth0 authentication service, and can be added to the GraphQL server that you're using together with the React application that you're building in this post.

To handle authorization, you first need to perform several actions in the Auth0 Dashboard:

- Add [permissions](https://auth0.com/docs/dashboard/guides/apis/add-permissions-apis) to the API that you've created in the GraphQL Server tutorial, which you can do by going to the [API](https://manage.auth0.com/#/apis/) page in the Auth0 dashboard and opening the Auth0 API you created for your GraphQL server. On this page, you need to create a new permission called `edit:events` on the _Permissions_ tab. Add "Edit events" as the description.

- On this same page, you need to make sure that both checkboxes for _RBAC Settings_ on the _Settings_ tab are checked. The first one enables Role-Based Access Control (RBAC) for the GraphQL server, and the second checkbox adds the permissions for the user to the JWT. Scroll down and click on _Save_.

- You need to add this permission to an [user role](https://auth0.com/docs/dashboard/guides/roles/add-permissions-roles) and add a user to this role. Adding a new user role can also be done by visiting the [Roles](https://manage.auth0.com/#/roles/) page. Here you must create a new role called `admin` using the _Create role_ button. After creating the role, you must add the `edit:events` permission and a user to it.

- Adding a `permission` and a user to this role can be done by clicking on the newly created role on the [Roles](https://manage.auth0.com/#/roles/) page. On the page that opens, you can add the `edit:events` permission to this role from the _Permissions_ tab and the user from the _Users_ tab, which must be the user you're using to log in to the application you've created in this post.

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

The `isTokenValid` function decoded the JWT, and inside the decoded JWT, the field `permissions` can be found. This field contains an array with permissions for this user, and if the permission `edit:events` that you've created before is present. If the user's JWT has this permission, it means that the user can edit the event, and the value for `canEdit` that is returned by the `event` query is true. Now, this query returns the field `canEdit` you must also add this field to the schema for this query in `src/schema.js`:

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

If a user is authenticated but doesn't have the correct permission to edit events, the `Form` component won't be displayed, making it unable for the user to edit the event details.

## Conclusion

In this tutorial, you've created a React application, that uses a GraphQL server to retrieve its data and Auth0 for authorization and authentication. You've learned how to set up a basic React application with routing, and Apollo to fetch data from a React server using queries and mutations. By using Auth0, authentication and authorization are added to the application, giving users different permissions for taking actions. The full code for this tutorial can be found on Github, as well as for the tutorial about creating a GraphQL server.