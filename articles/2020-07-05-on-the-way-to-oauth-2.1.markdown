---
layout: post
title: "On the way to OAuth 2.1 with the OAuth 2.0 Security Best Current Practice"
description: "The OAuth 2.0 Security Best Current Practices, which will be incorporated in OAuth 2.1, bring some innovations especially for web developers: The implicit flow is no longer be used. Instead, Code Flow with PKCE is recommended. Refresh tokens are now also permitted in the browser under certain cases."
date: "2020-07-05 08:30"
author:
  name: "Manfred Steyer"
  url: "ManfredSteyer"
  mail: "manfred.steyer@softwarearchitekt.at"
  avatar: "https://twitter.com/ManfredSteyer/profile_image?size=original"
related:
- 2017-11-15-an-example-of-all-possible-elements
---

**TL;DR:** The OAuth 2.0 Security Best Current Practice which will be incorporated in OAuth 2.1 brings some innovations, especially for single-page applications. It proposes authorization code flow with PKCE instead of implicit flow and allows refresh tokens. This article shows how to leverage those ideas by using an Angular application and Auth0. The example can be found [here](https://github.com/manfredsteyer/auth0-demo).

##  On the way to OAuth 2.1 with the OAuth 2.0 Security Best Current Practice
No serious business application can work without authentication and authorization. After all, the application must know who it is dealing with and which rights the current user has. A very flexible solution for this is the use of security tokens. They allow connecting to various identity solutions such as Active Directory and are the basis for the implementation of single sign-on.

To work with all possible identity solutions, we need a well-defined protocol. For several years now, OAuth 2.0 and OpenId Connect (OIDC) have been the de facto standards for this.

The [OAuth 2.0 Security Best Current Practice](https://tools.ietf.org/html/draft-ietf-oauth-security-topics-15), which will be incorporated in OAuth 2.1, bring some innovations especially for web developers: The implicit flow is no longer be used. Instead, Authorization Code Flow with PKCE is recommended. Refresh tokens are now also permitted in the browser under certain cases.

This article discusses these innovations and shows both the background and the consequences with a focus on single-page applications. To round it off, it illustrates the aspects discussed using an Angular application that leverages Auth0 for authentication and authorization.

The [example](https://github.com/manfredsteyer/auth0-demo) used for this can be found in [my GitHub account](https://github.com/manfredsteyer/auth0-demo). 

## Overview of OAuth 2.0 and OIDC

The principle behind OAuth 2.0 and OIDC is very easy to visualize with the so-called implicit flow. It is a simple variant that was originally designed for single-page applications. The flow begins by redirecting the user to an authorization server:

![OAuth 2.0 and OIDC from a high-level perspective](auth_01.png)

This server knows the individual accounts and prompts the user to log on. The type of authentication is an implementation detail that is not specified by the standard. Usernames and passwords or social login -- e. g. login with Facebook etc. -- are often used here. Besides this, the use of Windows authentication is also possible within Windows domains. In this case, the authorization server recognizes the current Windows user without further interaction.

The authorization server then issues a so-called access token as well as an id token (identity token) and redirects the user back to the client which is an Angular solution in our case.

The id token specified by OIDC informs the client about the current user. It contains key/value pairs -- so-called claims -- that describe the user. Some keys are standardized by OIDC. Examples are _given_name_, _family_name_, or _email_.

By definition, the id token is a JSON Web Token (JWT). To put it bluntly, it is a BASE64-encoded JSON document. Every client can easily parse and interpret it.

The access token, on the other hand, is defined by OAuth 2.0 and gives the client access to the backend, which the two standards refer to as a resource server. The client includes the token in an HTTP header. As the token is sensitive information, HTTPS must be used here. However, this is generally the case with OAuth 2.0 and OIDC.

After the resource server has successfully validated the access token, it finds claims in it that refer to the user. Digital signatures are often used to find out whether the access token comes from a trusted authorization server. Alternatively, the resource server can contact the authorization server to find out whether the access token is (still) valid. The former is easier and the latter is a little more secure because it constantly checks whether the user still has access.

A common middle ground is the use of short-lived access tokens. As described below, the client must regularly retrieve a new token for this type of game and it is checked whether the user still has the necessary permissions.

In contrast to the id token, the format of the access token is not specified. It can have any structure and is not necessarily legible for the client.

## OAuth 2.0 Security Best Current Practice

The implicit flow summarized in the last section has been the standard solution in single-page applications for a long time. The [OAuth 2.0 Security Best Current Practice](https://tools.ietf.org/html/draft-ietf-oauth-security-topics-15), on which the OAuth Working Group is currently working, changes this. This document paves the way to OAuth 2.1 and suggests that implicit flow is generally no longer used.

However, if you use the implicit flow in existing solutions, you do not have to panic: If you use it carefully with common best practices, you will withstand the known attacks. Some of these best practices are even mandated by OIDC and trustworthy libraries and authorization servers should take them into account.

One disadvantage of implicit flow is that the authorization server sends the tokens to the client using a redirect with hash parameters. These end up in the browser history but also in the logs of the authorization server, from where an attacker can steal them. Of course, the browser history can be overwritten with JavaScript and the logging behavior of the server can also be adjusted. To do this, however, we have to think about further steps.

To make this, together with many other established best practices for securing the implicit flow, unnecessary, the document mentioned suggests a different flow. This is called an authorization code flow. In combination with the PKCE (Proof Key for Code Exchange) [PKCE] extension, this flow is currently seen as the safest variant for the implementation of OAuth 2.0 and OIDC in the browser.

## Authorization Code Flow and PKCE

Authorization Code Flow provided the client only with an access code via the redirect instead of the actual token:

![Authorization Code Flow and PKCE](./Auth_02.png)

The authorization server also sends it to the client in the form of an URL parameter, but since the code is comparatively insensitive, this is fine. PKCE also ensures that an attacker cannot benefit from a stolen access code. For this purpose, this standard defines that the client sends the hash value of a random string to the authorization server when the original request is made. The random string itself is called PKCE Verifier.

If the client then wants to exchange the access code for the token, it must be sent along with the verifier:

![Exchange access code for tokens](./auth_03.png)

The authorization server now checks whether the verifier matches the previously sent hash value. If this is the case, the authorization server can be sure that it is communicating with the original client and not with an attacker who stole the access code. The answer includes the tokens. Since the access code is no longer exchanged via redirects but via a direct AJAX-based encrypted HTTPS connection, an attacker can no longer steal the tokens so easily.

## Token Refresh

Tokens usually have a short lifespan because if an attacker steals them -- for example via XSS -- they should not be able to work with them for long. A lifespan of 10 minutes is therefore not uncommon. A new token must, therefore, be requested every 10 minutes and this opportunity is also used to check whether the user has been blocked in the meantime.

Of course, this should be done without user interaction, because the user does not want to enter their password again every 10 minutes. The safest way to do this is to use HTTP-only cookies that cannot be stolen using XSS. Unfortunately, there is currently no standard-compliant option for using these cookies without redirecting to the authorization server. However, redirecting the user means they lose their state within the SPA. 

Workarounds for this included the usage of iframes and popups. While these solutions might work, they are obviously not that easy to implement and reliable.

A somewhat simpler option is to use refresh tokens:

![Refresh Tokens](./auth_04.png)

The authorization server issues the refresh token together with the access and id token and the client uses it to request additional tokens if required. Since an attacker can steal refresh tokens using XSS, in contrast to HTTP-only cookies, both OAuth 2.0 and OIDC prohibit their use in the browser.

This Best Practices document, however, deviates from this limitation, if there are some prerequisites. These include the following:

- The possible dangers must be assessed as part of a risk assessment. In concrete terms, this means that -- first and foremost -- measures must be taken to avoid XSS.
- Each refresh token can only be used once. If necessary, a new one must be issued afterward. This is called **refresh token rotation**. Also, if the same refresh token is used several times, all users who have used it should be logged out as in this case the authorization server cannot determine which one was an attacker.

There are some alternatives to refresh token rotation which, however, don't seem to be practical in most cases. Details can be found [here](https://tools.ietf.org/html/draft-ietf-oauth-security-topics-15#section-4.12.2).

Also, it's recommended to limit the lifetime of the refresh token. An easy way to accomplish that is to invalidate it when the user logs out.


## Setting up Auth0 as Your Authorization Server

Now, let's see all this in practice. For this, configure an application in your Auth0 account:

![Application configured in Auth0](auth0_01.png)

Make a note of the assigned ``Domain`` and the ``Client ID`` for later. You need it when configuring the Angular client.

Also, make sure your application is configured as a single page application:

![Configure your application as a SPA](auth0_02.png)

Define valid redirection URLs. Those are URLs of your application, Auth0 is allowed to send an access code to. This prevents attackers from redirecting the user alongside an access code to their own web site. Also, activate CORS for your application:

![Valid Redirection URL and CORS settings](auth0_03.png)

For the sake of brevity, some additional descriptions have been remove from this screenshot as well as from some of the subsequent ones.

As you see in the shown screenshot, our example assumes that the application is served from https://localhost:4200.

Besides this, activate the above discussed token rotation and make sure both, the id token and the refresh token are short living:

![Activating token rotation](auth0_04.png)

As we don't want the refresh token to outlive the user session for security reasons discussed above, set both lifetimes to the same value. This is not an issue because token rotation gives us a new refresh token alongside the refreshed access and id token. 

Within the advanced application settings at the end of the configuration page, you can activate Authorization Code Flow and the usage of refresh tokens:

![Allowed grant types](auth0_05.png)

Also, for getting a proper access token allowing us to access a resource server, configure an API in your Auth0 configuration console:

![Configured API](auth0_06.png)

Here, take note of the assigned identifier. You will need it when configuring the client. 

Besides, make sure the access token issued for your API is short living:

![Short living access token](auth0_07.png)


## Implementation in Angular

After configuring the application and the API in Auth0, we can implement our Angular client. For leveraging authorization code flow and PKCE, I use the _angular-oauth2-oidc_ library. It is widely used and has been tested with three different authorization servers to ensure that there is no over-adaptation to a specific manufacturer. Besides the Auth0, this is the .NET-based IdentityServer and Keycloak which is implemented in JAVA. It is also certified by the OpenId Connect Foundation, which leads to further trust.

After installation with npm (``npm install angular-oauth2-oidc``) it can be imported into the main module of the application:

```typescript
@NgModule ({
  imports: [
    BrowserModule,
    HttpClientModule,
    OAuthModule.forRoot ({
      resourceServer: {
        sendAccessToken: true,
        allowedUrls: ['https://www.angular.at/api/']
      }
    }),
  ],
  […]
})
export class AppModule {}
```

The ``sendAccessToken`` parameter specifies that the library should append the accessed token each time an Angular HTTP call is made. Under the covers, ``angular-oauth2-oidc`` uses an ``HttpInterceptor`` for this. To prevent sending the token accidentally to a wrong API, those who are allowed to receive it must be registered here. A prefix is ​​sufficient for this. In the case under consideration, all APIs whose URLs begin with https://www.angular.at/api/ receive the token.

To provide the library with key data such as the URL of the authorization server, a configuration object must be created:

```typescript
import { AuthConfig } from 'angular-oauth2-oidc';

export const authConfig: AuthConfig = {

    // Your Auth0 app's domain
    // Important: Don't forget to start with https://
    //  AND the trailing slash!
    issuer: 'https://dev-g-61sdfs.eu.auth0.com/',

    // The app's clientId configured in Auth0
    clientId: 'opHt1Tkt9E9fVQTZPBVF1tHVhjrxvyVX',

    // The app's redirectUri configured in Auth0
    redirectUri: window.location.origin,

    // Scopes ("rights") the Angular application wants get delegated
    scope: 'openid profile email offline_access',

    // Using Authorization Code Flow 
    // (PKCE is activated by default for authorization code flow)
    responseType: 'code',

    // Your Auth0 account's logout url
    // Derive it from your application's domain
    logoutUrl: 'https://dev-g-61sdfs.eu.auth0.com/v2/logout',

    customQueryParams: {
        // API identifier configured in Auth0
        audience: 'http://www.angular.at/api'
    },

};
```

The ``redirectUri`` mus be registered for the application with the configured ``clientId`` in Auth0. Otherwise, the authorization server must reject the request for security reasons, hence, in this case, an attacker could pretend to be a particular client.

The scopes ``openid``, ``profile`` and ``email`` are defined by OIDC and describe the information that the application wants to get about the user. In the simplest case, only ``openid`` is requested. In this case, the application only receives the user's ID. Behind ``profile`` are profile information such as first name or last name and ``email`` -- unsurprisingly -- the user's email address but also whether this address has already been verified by a test email.

The scope ``offline_access`` indicates that the application wants a refresh token.

It is also important at this point that the scopes do not reflect the rights of the user, but only those rights that the user delegates to the application.

To ensure that the library uses the configuration object, it must be set when the program starts:

```typescript
@Component ({…})
export class AppComponent {

  constructor (private oauthService: OAuthService) {
    oauthService.configure (authConfig);
    oauthService.setupAutomaticSilentRefresh ();
    oauthService.loadDiscoveryDocumentAndLogin().then([...]);
  }

}
```

With ``setupAutomaticSilentRefresh``, the application specifies that the tokens should be refreshed automatically when its lifetime is over.

The ``loadDiscoveryDocumentAndLogin`` method loads additional configuration data from the authorization server. This process, known as discovery, is standardized by OIDC. Also, it uses authorization code flow and PKCE to receive the tokens discussed.

To retrieve some information about the logged-in user, the ``OAuthService`` provides a ``getIdentityClaims`` method. It returns the claims found in the id token:

```typescript
const claims = this.oauthService.getIdentityClaims ();
this.userName = ['given_name'];
```

## Working With the token

Now let's call an API and find out whether the library automatically sends the access token via the HTTP authorization header:

![HTTP header with access token](token_01.png)

By default, the library stores the tokens in the session storage, although this can also be changed via the configuration. If you are curious, you can read them using the browser's dev tools.

The page https://jwt.io offers a very simple way to analyze tokens that use the JWT format. For example, the received access token looks like this:

![Decoded access token on jwt.io](./token_02.png)

It consists of three parts: The header shown in red contains general information such as the crypto algorithms used for the digital signature and public keys. In addition to technical information such as the expiry date (``exp``), the payload contains the user's id (``sub``), the grated scopes, and it's audience (``aud``) array points to our API. 

Besides this, ``aud`` also points to Auth0's userinfo endpoint which allows us to receive data about the logged-in user. This is especially useful when the id token does not contain all the claims we need.

The id token looks similar:

![Decoded id token on jwt.io](./token_03.png)

However, its payload contains more information about the user like ``given_name``, ``family _name``, ``email``, and ``email_verified``.


## Logging Out the User 

When the user logs out, it's important to revoke the refresh token to prevent attackers from using it to get access tokens for impersonating the user. For the same reason, the user's session at Auth0 needs to be ended.

For this, the application shown here uses the ``revokeTokenAndLogout`` method:

```typescript
this.oauthService.revokeTokenAndLogout({
  client_id: this.oauthService.clientId,
  returnTo: this.oauthService.redirectUri
}, true);
```

Auth0's logout endpoint expects the app's ``clientId``. Also, one might send over a parameter ``returnTo`` with an URL the user shall be redirected to after logging them out. The 2nd parameter makes the method to ignore CORS-related issues during the token revocation. Once Auth0's token revocation endpoint supports CORS, we can skip this parameter.

## Testing Your Application With HTTPS

As OAuth 2 and OIDC depend upon HTTPS, Auth0 only allows to register HTTPS-based endpoints. To avoid issues with mixed content (HTTP and HTTPS) in your browser, serve your Angular application using the ``--ssl`` flag:

```
ng serve --ssl -o
```

This makes the CLI providing the application via https://localhost:4200. However, the browser does not trust the certificate used by the CLI. Hence, you'll get a warning. You can bypass this warning or create an own certificate trusted by your machine for the CLI. Both is described in this [blog article](https://medium.com/@rubenvermeulen/running-angular-cli-over-https-with-a-trusted-certificate-4a0d5f92747a).


## Conclusion and outlook

OAuth 2.0 and OIDC are simple. However, the devil is in the details and even large corporations like Facebook opened the door to attackers by deviating a little from the standard. That is why it is advisable to use proven solutions instead of implementing these standards by hand.

The new Best Practices document, which is to be incorporated in OAuth 2.1, also makes handling OAuth 2.0 and OIDC easier: The implicit flow is eliminated and many best practices for preventing attacks are therefore unnecessary. Also, refresh tokens make retrieving new tokens during the user's session straightforward and stable.

Since refresh tokens represent extremely sensitive information and can be stolen using XSS attacks, strategies must be used here to prevent XSS. Angular does a very good job of coding expenses here by default. However, it is also important to prevent npm packages and third-party scripts from causing trouble in the application. NPM's audit information and commercial library validation products help as well as manual audits conducted by security experts.

The most secure way to prevent the theft of refresh tokens is to use HTTP-only cookies. Although this is already possible today, it is unfortunately not standardized by OAuth 2 or OIDC. The standards would have to be expanded especially for an AJAX-based token refresh using HTTP-only cookies with refresh tokens. Since such an extension would significantly improve the security of SPAs, it is at the top of my wish list.

___

Manfred Steyer is a trainer and consultant with a focus on Angular, a Google Developer Expert, and a Trusted Collaborator in the Angular team. He writes for O'Reilly, the German Java Magazine, and Heise Developer. Also, he regularly speaks at conferences. At [www.angulararchitects.io](https://www.angulararchitects.io/), he offers Angular training, workshops, and consultancy.