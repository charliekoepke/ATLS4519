# Creative Project

> Choose a Node.js framework and npm package to learn about, and build a creative project that demonstrates the technologies usage and potential. Make it academic, artistic, and creative! Include a comprehensive "How To" as well, so other people can use and learn.

The Node.js I decided to use is [koa.js](https://koajs.com/). They have all of their functions and how to use them included on their website, so my project aimed to illustrate those functions in a more artistic and creative way.

# koa.js

## Introduction

Koa is a new web framework designed by the team behind Express, which aims to be a smaller, more expressive, and more robust foundation for web applications and APIs. By leveraging async functions, Koa allows you to ditch callbacks and greatly increase error-handling. Koa does not bundle any middleware within its core, and it provides an elegant suite of methods that make writing servers fast and enjoyable.

---

## Installation

Koa requires **node v7.6.0** or higher for ES2015 and async function support.

You can quickly install a supported version of node with your favorite version manager:
```
$ nvm install 7
$ npm i koa
$ node my-koaapp.js
```

## Application

A Koa application is an object containing an array of middleware functions which are composed and executed in a stack-like manner upon request. Koa is similar to many other middleware systems that you may have encountered such as Ruby's Rack, Connect, and so on - however a key design decision was made to provide high level "sugar" at the otherwise low-level middleware layer. This improves interoperability, robustness, and makes writing middleware much more enjoyable.

This includes methods for common tasks like content-negotiation, cache freshness, proxy support, and redirection among others. Despite supplying a reasonably large number of helpful methods Koa maintains a small footprint, as no middleware are bundled.

The obligatory hello world application:
```
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => {
    ctx.body = 'Hello World';
});

app.listen(3000);
```

### Cascading

Koa middleware cascade in a more traditional way as you may be used to with similar tools - this was previously difficult to make user friendly with node's use of callbacks. However with async functions we can achieve "true" middleware. Contrasting Connect's implementation which simply passes control through series of functions until one returns, Koa invoke "downstream", then control flows back "upstream".

The following example responds with "Hello World", however first the request flows through the `x-response-time` and `logging` middleware to mark when the request started, then yields control through the response middleware. When a middleware invokes `next()` the function suspends and passes control to the next middleware defined. After there are no more middleware to execute downstream, the stack will unwind and each middleware is resumed to perform its upstream behaviour.
```
const Koa = require('koa');
const app = new Koa();

// logger 

app.use(async (ctx, next) => {
    await next();
    const rt = ctx.response.get('X-Response-Time');
    console.log(`${ctx.method} ${ctx.url} - ${rt}`);

});

// x-response-time

app.use(async (ctx, next) => {
    const start = Date.now();
    await next();
    const ms = Date.now() - start;
    ctx.set('X-Response-Time', `${ms}ms`);
});

// response

app.use(async ctx => {
    ctx.body = 'Hello World';
});

app.listen(3000);
```

### Settings

Application settings are properties on the `app` instance, currently the following are supported:
- `app.env` defaulting to the **NODE_ENV** or "development"
- `app.keys` array of signed cookie keys
- `app.proxy` when true proxy header fields will be trusted
- `app.subdomainOffset` offset of `.subdomains` to ignore, default to 2
- `app.proxyIpHeader` proxy ip header, default to `X-Forwarded-For`
- `app.maxIpsCount` max ips read from proxy ip header, default to 0 (means infinity)

You can pass the settings to the constructor:
```
const Koa = require('koa');
const app = new Koa({ proxy: true });
```
or dynamically:
```
const Koa = require('koa');
const app = new Koa();
app.proxy = true;
```

---

### app.listen(...)

A Koa application is not a 1-to-1 representation of an HTTP server. One or more Koa applications may be mounted together to form larger applications with a single HTTP server.

Create and return an HTTP server, passing the given arguments to `Server#listen()`. These arguments are documented on [nodejs.org](https://nodejs.org/api/http.html#http_server_listen_port_hostname_backlog_callback). The following is a useless Koa application bound to port `3000`:
```
const Koa = require('koa');
const app = new Koa();
app.listen(3000);
```
The `app.listen(...)` method is simply sugar for the following:
```
const http = require('http');
const Koa = require('koa');
const app = new Koa();
http.createServer(app.callback()).listen(3000);
```
This means you can spin up the same application as both HTTP and HTTPS or on multiple addresses:
```
const http = require('http');
const https = require('https');
const Koa = require('koa');
const app = new Koa();
http.createServer(app.callback()).listen(3000);
https.createServer(app.callback()).listen(3001);
```

### app.callback()

Return a callback function suitable for the `http.createServer()` method to handle a request. You may also use this callback function to mount your Koa app in a Connect/Express app.

### app.use(function)

Add the given middleware function to this application. app.use() returns this, so is chainable.
```
app.use(someMiddleware);
app.use(someOtherMiddleware);
app.listen(3000);
```
Is the same as
```
app.use(someMiddleware)
.use(someOtherMiddleware);
.listen(3000);
```
See [Middleware](https://github.com/koajs/koa/wiki#middleware) for more information.

### app.keys=

Set signed cookie keys.

These are passed to [KeyGrip](https://github.com/crypto-utils/keygrip), however you may also pass your own `KeyGrip` instance. For example the following are acceptable:
```
app.keys = ['OEK5zjaAMPc3L6iK7PyUjCOziUH3rsrMKB9u8H07La1SkfwtuBoDnHaaPCkG5Brg', 'MNKeIebviQnCPo38ufHcSfw3FFv8EtnAe1xE02xkN1wkCV1B2z126U44yk2BQVK7'];
app.keys = new KeyGrip(['OEK5zjaAMPc3L6iK7PyUjCOziUH3rsrMKB9u8H07La1SkfwtuBoDnHaaPCkG5Brg', 'MNKeIebviQnCPo38ufHcSfw3FFv8EtnAe1xE02xkN1wkCV1B2z126U44yk2BQVK7'], 'sha256');
```
For security reasons, please ensure that the key is long enough and random.

These keys may be rotated and are used when signing cookies with the `{ signed: true }` option:
```
ctx.cookies.set('name', 'tobi', { signed: true });
```

### app.context

`app.context` is the prototype from which `ctx` is created. You may add additional properties to `ctx` by editing `app.context`. This is useful for adding properties or methods to `ctx` to be used across your entire app, which may be more performant (no middleware) and/or easier (fewer `require()`s) at the expense of relying more on `ctx`, which could be considered an anti-pattern.

For example, to add a reference to your database from `ctx`:
```
app.context.db = db();

app.use(async ctx => {
    console.log(ctx.db);
});
```
Note:

- Many properties on `ctx` are defined using getters, setters, and `Object.defineProperty()`. You can only edit these properties (not recommended) by using `Object.defineProperty()` on `app.context`. See https://github.com/koajs/koa/issues/652.
- Mounted apps currently use their parent's `ctx` and settings. Thus, mounted apps are really just groups of middleware.

### Error Handling

By default outputs all errors to stderr unless `app.silent` is `true`. The default error handler also won't output errors when `err.status` is `404` or `err.expose` is `true`. To perform custom error-handling logic such as centralized logging you can add an "error" event listener:
```
app.on('error', err => {
    log.error('server error', err)
});
```
If an error is in the req/res cycle and it is *not* possible to respond to the client, the `Context` instance is also passed:
```
app.on('error', (err, ctx) => {
  log.error('server error', err, ctx)
});
```
When an error occurs *and* it is still possible to respond to the client, aka no data has been written to the socket, Koa will respond appropriately with a 500 "Internal Server Error". In either case an app-level "error" is emitted for logging purposes.

---

## Context

A Koa Context encapsulates node's `request` and `response` objects into a single object which provides many helpful methods for writing web applications and APIs. These operations are used so frequently in HTTP server development that they are added at this level instead of a higher level framework, which would force middleware to re-implement this common functionality.

A `Context` is created *per* request, and is referenced in middleware as the receiver, or the `ctx` identifier, as shown in the following snippet:
```
app.use(async ctx => {
    ctx; // is the Context
    ctx.request; // is a Koa Request
    ctx.response; // is a Koa Response
});
```
Many of the context's accessors and methods simply delegate to their `ctx.request` or `ctx.response` equivalents for convenience, and are otherwise identical. For example `ctx.type` and `ctx.length` delegate to the response object, and `ctx.path` and `ctx.method` delegate to the request.

### API

`Context` specific methods and accessors.

### ctx.req

Node's `request` object.

### ctx.res

Node's `response` object.

Bypassing Koa's response handling is **not supported**. Avoid using the following node properties:
- `res.statusCode`
- `res.writeHead()`
- `res.write()`
- `res.end()`

### ctx.request

A Koa `Request` object.

### ctx.response

A Koa `Response` object.

### ctx.state

The recommended namespace for passing information through middleware and to your frontend views.
```
ctx.state.user = await User.find(id);
```

### ctx.app

Application instance reference.

### ctx.app.emit

Koa applications extend an internal [EventEmitter](https://nodejs.org/dist/latest-v11.x/docs/api/events.html). `ctx.app.emit` emits an event with a type, defined by the first argument. For each event you can hook up "listeners", which is a function that is called when the event is emitted. Consult the error handling docs for more information.

### ctx.cookies.get(name, [options])

Get cookie `name` with `options`:
- `signed` the cookie requested should be signed
Koa uses the [cookies](https://github.com/pillarjs/cookies) module where options are simply passed.

### ctx.cookies.set(name, value, [options])

Set cookie `name` to `value` with `options`:

- `maxAge`: a number representing the milliseconds from `Date.now()` for expiry.
- `expires`: a `Date` object indicating the cookie's expiration date (expires at the end of session by default).
- `path`: a string indicating the path of the cookie (`/` by default).
- `domain`: a string indicating the domain of the cookie (no default).
- `secure`: a boolean indicating whether the cookie is only to be sent over HTTPS (`false` by default for HTTP, `true` by default for HTTPS). [Read more about this option](https://github.com/pillarjs/cookies#secure-cookies).
- `httpOnly`: a boolean indicating whether the cookie is only to be sent over HTTP(S), and not made available to client JavaScript (`true` by default).
- `sameSite`: a boolean or string indicating whether the cookie is a "same site" cookie (`false` by default). This can be set to '`strict`', '`lax`', '`none`', or `true` (which maps to '`strict`').
- `signed`: a boolean indicating whether the cookie is to be signed (`false` by default). If this is true, another cookie of the same name with the `.sig` suffix appended will also be sent, with a 27-byte url-safe base64 SHA1 value representing the hash of *cookie-name=cookie-value* against the first [Keygrip](https://www.npmjs.com/package/keygrip) key. This signature key is used to detect tampering the next time a cookie is received.
- `overwrite`: a boolean indicating whether to overwrite previously set cookies of the same name (`false` by default). If this is true, all cookies set during the same request with the same name (regardless of path or domain) are filtered out of the Set-Cookie header when setting this cookie.
Koa uses the [cookies](https://github.com/pillarjs/cookies) module where options are simply passed.

### ctx.throw([status], [msg], [properties])

Helper method to throw an error with a `.status` property defaulting to `500` that will allow Koa to respond appropriately. The following combinations are allowed:
```
ctx.throw(400);
ctx.throw(400, 'name required');
ctx.throw(400, 'name required', { user: user });
```
For example `ctx.throw(400, 'name required')` is equivalent to:
```
const err = new Error('name required');
err.status = 400;
err.expose = true;
throw err;
```

Note that these are user-level errors and are flagged with `err.expose` meaning the messages are appropriate for client responses, which is typically not the case for error messages since you do not want to leak failure details.

You may optionally pass a `properties` object which is merged into the error as-is, useful for decorating machine-friendly errors which are reported to the requester upstream.
```
ctx.throw(401, 'access_denied', { user: user });
```
Koa uses [http-errors](https://github.com/jshttp/http-errors) to create errors. status should only be passed as the first parameter.

### ctx.assert(value, [status], [msg], [properties])

Helper method to throw an error similar to `.throw()` when `!value`. Similar to node's [assert()](https://nodejs.org/api/assert.html) method.
```
ctx.assert(ctx.state.user, 401, 'User not found. Please login!');
```
Koa uses [http-assert](https://github.com/jshttp/http-assert) for assertions.

### ctx.respond

To bypass Koa's built-in response handling, you may explicitly set `ctx.respond = false;`. Use this if you want to write to the raw `res` object instead of letting Koa handle the response for you.

Note that using this is **not** supported by Koa. This may break intended functionality of Koa middleware and Koa itself. Using this property is considered a hack and is only a convenience to those wishing to use traditional `fn(req, res)` functions and middleware within Koa.

### Request Aliases

The following accessors and alias Request equivalents:

- `ctx.header`
- `ctx.headers`
- `ctx.method`
- `ctx.method=`
- `ctx.url`
- `ctx.url=`
- `ctx.originalUrl`
- `ctx.origin`
- `ctx.href`
- `ctx.path`
- `ctx.path=`
- `ctx.query`
- `ctx.query=`
- `ctx.querystring`
- `ctx.querystring=`
- `ctx.host`
- `ctx.hostname`
- `ctx.fresh`
- `ctx.stale`
- `ctx.socket`
- `ctx.protocol`
- `ctx.secure`
- `ctx.ip`
- `ctx.ips`
- `ctx.subdomains`
- `ctx.is()`
- `ctx.accepts()`
- `ctx.acceptsEncodings()`
- `ctx.acceptsCharsets()`
- `ctx.acceptsLanguages()`
- `ctx.get()`

### Response Aliases

The following accessors and alias Response equivalents:

- `ctx.body`
- `ctx.body=`
- `ctx.status`
- `ctx.status=`
- `ctx.message`
- `ctx.message=`
- `ctx.length=`
- `ctx.length`
- `ctx.type=`
- `ctx.type`
- `ctx.headerSent`
- `ctx.redirect()`
- `ctx.attachment()`
- `ctx.set()`
- `ctx.append()`
- `ctx.remove()`
- `ctx.lastModified=`
- `ctx.etag=`

---

## Request

### API

### request.header

### request.header=

### request.headers

### request.headers=

### request.method

### request.method=

### request.length

### request.url

### request.url=

### request.originalUrl

### request.origin

### request.href

### request.path

### request.path=

### request.querystring

### request.querystring=

### request.search

### request.search=

### request.host

### request.hostname

### request.URL

### request.type

### request.charset

### request.query

### request.query=

### request.fresh

### request.stale

### request.protocol

### request.secure

### request.ip

### request.ips

### request.subdomains

### request.is(types...)

### Content Negotiation

### request.accepts(types)

### request.acceptsEncodings(encodings)

### request.acceptsCharsets(charsets)

### request.acceptsLanguages(lang)

### request.idempotent

### request.socket

### request.get(field)

---

## Response

### API

### response.header

### response.headers

### response.socket

### response.status

### response.status=

### response.message

### response.message=

### response.length=

### response.length

### repsonse.body

### response.body=

### String

### Buffer

### Stream

### Object

### response.get(field)

### response.has(field)

### response.set(field, value)

### response.append(field, value)

### response.set(fields)

### response.remove(fields)

### response.type

### response.type=

### response.is(types...)

### response.redirect(url,[alt])

### response.attachment([filename], [options])

### response.headerSent

### response.lastModified

### response.lastModified=

### response.etag=

### response.vary(field)

### response.flushHeaders()

---

## Sponsor

---

## Links

---

## Sources

[koa.js website](https://koajs.com/)