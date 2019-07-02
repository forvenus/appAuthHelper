# Build an Identity Proxy for your JavaScript Apps with Service Workers

[Service Workers](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API) are a relatively-new feature within the browser platform. At their core, they offer a separate JavaScript execution context that has access to intercept (and potentially modify) all network traffic generated by the primary execution context. Essentially, they are a proxy layer that exists in the browser between the main application code and the network endpoints which it calls.

The primary use-case described for Service Workers is to handle things such as off-line access using sophisticated caching strategies. However, service workers are a generic browser feature - they are not limited to just the uses they were originally designed for.

One exciting use of service workers doesn't have anything to do with caching - instead, it is relevant to applications which are operating as an OAuth 2.0 client. The main job of a OAuth 2.0 client is to make requests to OAuth 2.0 resource server endpoints with an [access token included as an authorization header](https://tools.ietf.org/html/rfc6750). This job can be done very easily with the help of a service worker "identity proxy".

The act of adding a bearer token to a XHR request is more complicated than it seems at first. You might think it is as simple as this:

```JavaScript
var deferred = $.ajax({
    url: "https://rs.example.com/me",
    headers: {
        "Authorization": "Bearer " + getAccessToken()
    }
});
```

However, there are challenges that this simplistic code does not address. For example, access tokens can expire; how should your code react to such an occurrence? According to the [bearer token specification](https://tools.ietf.org/html/rfc6750#section-3.1), expired tokens are reported in a 401 error response using the "WWW-Authenticate Response Header Field"; in particular, the "invalid_token" error code. Per that spec:

> The client MAY request a new access token and retry the protected resource request.

In order to do that, you would have to add code that checks for the expired token error, fetches a new access token (possibly using a refresh token), and then retries the original request with the new access token. What might that do to your previously-simple code? Here's a best-case example of what that might look like:

```JavaScript
var deferred = $.Deferred(),
    requestDetails = function (token) {
        return {
            url: "https://rs.example.com/me",
            headers: {
                "Authorization": "Bearer " + getAccessToken()
            }
        };
    };

$.ajax(requestDetails()).then(deferred.resolve, function (jqXHR) {
    if (getAuthHeaderError(jqXHR) === "invalid_token") {
        refreshAccessToken().then(function () {
            $.ajax(requestDetails())
                .then(deferred.resolve, deferred.reject);
        }, deferred.reject);
    } else {
        deferred.reject(jqXHR);
    }
});
```

That is a lot of extra code; suddenly your XHR request isn't so simple. Adding that retry logic everywhere your application makes a call to a resource server is not so appealing. Even wrapping all of that logic up into a shared function would mean that your application would have to use that shared function instead of its normal manner for making network requests. Contrast with this version:

```JavaScript
var deferred = $.ajax({
    url: "https://rs.example.com/me"
});
```

Now this is simple! Note the complete lack of access token reference. The application code does not have to worry about it at all. Instead, it can rely on the service worker (acting as an identity proxy) to intercept this outgoing request and alter it before it leaves the browser. The service worker will handle all details related to token selection, usage, renewal, etc... The result returned to the application will be whatever the service worker produces. In the case when it was able to retry a request, it will be the most recent result.

It is important to note that a service worker is only able to intercept network requests from JavaScript code that is hosted in the same origin as the service worker. This is actually a security benefit for the identity proxy use-case; there is no risk of a cross-site request forgery (CSRF) attack which exploits things like cookies always being present regardless of the request origin.

In addition to handling token renewal, a server worker identity proxy can help you properly implement best practices for working with multiple resource servers. According to good advice found in the [OAuth 2.0 best current practice](https://tools.ietf.org/id/draft-ietf-oauth-security-topics-10.html#aud_restriction), you should use specific access tokens for each resource server your client requests. The goal is to limit the exposure of scopes, so that one potentially-misbehaving resource server could not take the token you give it and use it to make requests a different server. To accomplish this goal, your client will have to acquire multiple, appropriately-scoped access tokens and add them to the correct resource server requests. Selecting the right token to add to each of your requests is additional complexity - don't burden your application code with it! Let the identity proxy handle it for you.

Here is how the service worker can do all of those things for your application code. Below is an annotated copy of a [real implementation](https://www.npmjs.com/package/appauthhelper) of a service worker operating as an identity proxy. Read through the code and comments to learn how it works:

```JavaScript
// This is how service workers intercept network request events -
// every network request made by the application code will trigger
// this event listener.
self.addEventListener("fetch", (event) => {

    // By the time this event listener is called, the service worker should
    // already have been configured to watch for requests to particular servers.
    if (self.appAuthConfig &&
        typeof self.appAuthConfig.resourceServers === "object" &&
        Object.keys(self.appAuthConfig.resourceServers).length) {

        // Check to see if this particular network request event
        // is to one of the configured resource servers.
        var resourceServer = Object.keys(self.appAuthConfig.resourceServers)
            .filter((rs) => event.request.url.indexOf(rs) === 0)[0];

        if (resourceServer) {
            // This is how service workers alter the response that is
            // presented to the application code - You can resolve the
            // promise passed into `respondWith` with any Response object.
            // https://developer.mozilla.org/en-US/docs/Web/API/Response
            event.respondWith(new Promise((resolve, reject) => {

                // Modify the current request by adding the appropriate
                // access token to it.
                self.addAccessTokenToRequest(event.request, resourceServer)
                    // Make the token-bearing request to the resource server.
                    .then((rsRequest) => fetch(rsRequest))
                    .then((resp) => {
                        // Check for the special-case error that we can possibly recover from.
                        if (!resp.ok && getAuthHeaderDetails(resp.headers)["error"] === "invalid_token") {
                            /*
                            From https://tools.ietf.org/html/rfc6750#section-3.1:
                            invalid_token
                                 The access token provided is expired, revoked, malformed, or
                                 invalid for other reasons.  The resource SHOULD respond with
                                 the HTTP 401 (Unauthorized) status code.  The client MAY
                                 request a new access token and retry the protected resource
                                 request.

                            We are going to follow the RFC's advice here and try to request a new
                            access token and retry the request.
                            */
                            let promise = self.waitForRenewedToken(resourceServer)
                                .then(() => self.addAccessTokenToRequest(event.request, resourceServer))
                                .then((freshRSRequest) => fetch(freshRSRequest));

                            // The act of obtaining a fresh token is done outside
                            // of the service worker. Use a MessageChannel to signal
                            // the main execution context that a fresh token is needed.
                            // https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel
                            self.messageChannel.postMessage({
                                "message":"renewTokens",
                                "resourceServer": resourceServer
                            });
                            return promise;
                        } else {
                            return resp;
                        }
                    })
                    .then(resolve, reject);
            }));
        }
    }
    return;
});
```

Though their names are fairly self-explanitory, the implementations of the functions `getAuthHeaderDetails`, `addAccessTokenToRequest` and `waitForRenewedToken` are available within the [source for this project](https://github.com/ForgeRock/appAuthHelper/blob/master/appAuthServiceWorker.js).

An important detail is worth mentioning regarding the service worker context - it only has access to a limited number of APIs, all of which must be asynchronous. For example, there is no access to the DOM. In an OAuth 2.0 environment, this can present a challenge - the typical means of obtaining access tokens is via browser redirection (user interaction is sometimes required). For this reason, only the specific concern of making token-bearing requests must be handled within the service worker. All other concerns (such as actually obtaining tokens from the authorization server) must be handled in the main execution context.

To address the coordination of these concerns, the most sensible approach is to define a [MessageChannel](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel) connection between the service worker context and the main application context. For example, by using this channel the service worker can notify the main context that it must attempt to renew an access token (an example of this type of message is in the code above). Likewise, the main execution context can use the same channel to post a message to the service worker to notify it when the fresh tokens are available.

This technique works best as part of a larger library that manages the whole OAuth 2.0 life-cycle. One such implementation is [ForgeRock's AppAuthHelper](https://www.npmjs.com/package/appauthhelper) library, which is itself built upon [AppAuth for JS](https://github.com/openid/AppAuth-JS). AppAuthHelper allows you to worry much less about the subtle and complex aspects of OAuth 2.0, and instead focus on the business logic your client application is interested in. Review the [README for AppAuthHelper](https://github.com/ForgeRock/appAuthHelper/blob/master/README.md) to learn more about how to use it for your application.

Save your JavaScript Apps from OAuth 2.0 complexities! The Identity Proxy is here to help.