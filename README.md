# Programmable HTTP proxy server for Node.js

[![npm version](https://badge.fury.io/js/proxy-chain.svg)](http://badge.fury.io/js/proxy-chain)
[![Build Status](https://travis-ci.org/apifytech/proxy-chain.svg)](https://travis-ci.org/apifytech/proxy-chain)

Node.js implementation of a proxy server (think Squid) with support for SSL, authentication, upstream proxy chaining
and custom HTTP responses.
The authentication and proxy chaining configuration is defined in code and can be dynamic.
Note that the proxy server only supports Basic authentication
(see [Proxy-Authorization](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Proxy-Authorization) for details).

For example, this package is useful if you need to use proxies with authentication
in the headless Chrome web browser, because it doesn't accept proxy URLs such as `http://username:password@proxy.example.com:8080`.
With this library, you can setup a local proxy server without any password
that will forward requests to the upstream proxy with password.
For this very purpose the package is used by the [Apify web scraping platform](https://www.apify.com).

To learn more about the rationale behind this package,
read [How to make headless Chrome and Puppeteer use a proxy server with authentication](https://medium.com/@jancurn/how-to-make-headless-chrome-and-puppeteer-use-a-proxy-server-with-authentication-249a21a79212).


## Run a simple HTTP/HTTPS proxy server

```javascript
const ProxyChain = require('proxy-chain');

const server = new ProxyChain.Server({ port: 8000 });

server.listen(() => {
    console.log(`Proxy server is listening on port ${8000}`);
});
```

## Run a HTTP/HTTPS proxy server with credentials and upstream proxy

```javascript
const ProxyChain = require('proxy-chain');

const server = new ProxyChain.Server({
    // Port where the server the server will listen. By default 8000.
    port: 8000,

    // Enables verbose logging
    verbose: true,

    // Custom function to authenticate proxy requests and provide the URL to chained upstream proxy.
    // It must return an object (or promise resolving to the object) with following form:
    // { requestAuthentication: Boolean, upstreamProxyUrl: String }
    // If the function is not defined or is null, the server runs in a simple mode.
    // Note that the function takes a single argument with the following properties:
    // * request  - An instance of http.IncomingMessage class with information about the client request
    //              (which is either HTTP CONNECT for SSL protocol, or other HTTP request)
    // * username - Username parsed from the Proxy-Authorization header. Might be empty string.
    // * password - Password parsed from the Proxy-Authorization header. Might be empty string.
    // * hostname - Hostname of the target server
    // * port     - Port of the target server
    // * isHttp   - If true, this is a HTTP request, otherwise it's a HTTP CONNECT tunnel for SSL
    //              or other protocols
    prepareRequestFunction: ({ request, username, password, hostname, port, isHttp }) => {
        return {
            // Require clients to authenticate with username 'bob' and password 'TopSecret'
            requestAuthentication: username !== 'bob' || password !== 'TopSecret',

            // Sets up an upstream HTTP proxy to which all the requests are forwarded.
            // If null, the proxy works in direct mode.
            upstreamProxyUrl: `http://username:password@proxy.example.com:3128`,
        };
    },
});

server.listen(() => {
  console.log(`Proxy server is listening on port ${server.port}`);
});
```

## Run a HTTP proxy server with custom responses

Custom responses allow you to override the response to a HTTP requests to the proxy, without contacting any target hoste.
For example, this is useful if you want to provide a HTTP proxy-style interface
to an external API or respond with some custom page to certain requests.
Note that this feature is only available for HTTP connections. That's because HTTPS
connections cannot be intercepted without access to target host's private key.

To provide a custom response, the result of the `prepareRequestFunction` function must
define the `customResponseFunction` property, which contains a function that generates the custom response.
The function is passed no parameters and it must return an object (or a promise resolving to an object)
with the following properties:

```javascript
{
  // Optional HTTP status code of the response. By default it is 200.
  statusCode: 200,

  // Optional HTTP headers of the response
  headers: {
    'X-My-Header': 'bla bla',
  }

  // Optional string with the body of the HTTP response
  body: 'My custom response',

  // Optional encoding of the body. If not provided, defaults to 'UTF-8'
  encoding: 'UTF-8',
}
```

Here is a simple example:

```javascript
const ProxyChain = require('proxy-chain');

const server = new ProxyChain.Server({
    port: 8000,
    prepareRequestFunction: ({ request, username, password, hostname, port, isHttp }) => {
        return {
            customResponseFunction: () => {
                return {
                    statusCode: 200,
                    body: `My custom response to ${request.url}',
                }
            },
        };
    },
});

server.listen(() => {
  console.log(`Proxy server is listening on port ${server.port}`);
});
```

## Closing the server

To shutdown the proxy server, call the `close([destroyConnections], [callback])` function. For example:

```javascript
server.close(true, () => {
  console.log('Proxy server was closed.');
});
```

The `closeConnections` parameter indicates whether pending proxy connections should be forcibly closed.
If the `callback` parameter is omitted, the function returns a promise.


## Helper functions

The package also provides several utility functions.


### `anonymizeProxy(proxyUrl, callback)`

Parses and validates a HTTP proxy URL. If the proxy requires authentication,
then the function starts an open local proxy server that forwards to the proxy.
The port is chosen randomly.

The function takes optional callback that receives the anonymous proxy URL.
If no callback is supplied, the function returns a promise that resolves to a String with
anonymous proxy URL or the original URL if it was already anonymous.


### `closeAnonymizedProxy(anonymizedProxyUrl, closeConnections, callback)`

Closes anonymous proxy previously started by `anonymizeProxy()`.
If proxy was not found or was already closed, the function has no effect
and its result if `false`. Otherwise the result is `true`.

The `closeConnections` parameter indicates whether pending proxy connections are forcibly closed.

The function takes optional callback that receives the result Boolean from the function.
If callback is not provided, the function returns a promise instead.

### `createTunnel(proxyUrl, targetHost, options, callback)`

Creates a TCP tunnel to `targetHost` that goes through a HTTP proxy server
specified by the `proxyUrl` parameter.

The result of the function is local endpoint in a form of `hostname:port`.
All TCP connections made to the local endpoint will be tunneled through the proxy to the target host and port.
For example, this is useful if you want to access a certain service from a specific IP address.

The tunnel should be eventually closed by calling the `closeTunnel()` function.

The `createTunnel()` function accepts an optional Node.js-style callback that receives the path to the local endpoint.
If no callback is supplied, the function returns a promise that resolves to a String with
the path to the local endpoint.

Example:

```javascript
const host = await createTunnel('http://bob:pass123@proxy.example.com:8000', 'service.example.com:356');
// Prints something like "localhost:56836"
console.log(host);
```

### `closeTunnel(tunnelString, closeConnections, callback)`

Closes tunnel previously started by `createTunnel()`.
The result value is `false` if the tunnel was not found or was already closed, otherwise it is `true`.

The `closeConnections` parameter indicates whether pending connections are forcibly closed.

The function takes an optional callback that receives the result of the function.
If the callback is not provided, the function returns a promise instead.

### `parseUrl(url)`

Calls Node.js's [url.parse](https://nodejs.org/docs/latest/api/url.html#url_url_parse_urlstring_parsequerystring_slashesdenotehost)
function and extends the resulting object with the following fields: `scheme`, `username` and `password`.
For example, for `HTTP://bob:pass123@example.com` these values are
`http`, `bob` and `pass123`, respectively.


### `redactUrl(url, passwordReplacement)`

Takes a URL and hides the password from it. For example:

```javascript
// Prints 'http://bob:<redacted>@example.com'
console.log(redactUrl('http://bob:pass123@example.com'));
```
