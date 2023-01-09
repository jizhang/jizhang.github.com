---
title: Mock API in Parcel Project
tags:
  - frontend
  - parcel
  - connect
categories: Programming
date: 2023-01-09 14:53:40
---


When developing a frontend application, usually we create mocks for backend API, so that after the API contract is settled down, front and backend engineers can work independently. There are several ways to accomplish this task, such as start a dedicated server and let the build tool serve as a proxy, or we can add middleware directly into the build tool's dev server, if applicable. Some tools can monkey patch the network calls to replace the response with mock data, and various unit testing tools provide their own way of mocking. In this article, I will focus on how to add middleware into [Parcel][1]'s dev server to respond with mock data for API calls.

![Parcel](images/parcel.png)

## API Proxy in Parcel's development server

Parcel provides a dev server and supports [API proxy][2] out of the box. Under the hood, it uses [connect][3] and [http-proxy-middleware][4] to redirect API calls to a different server. It also provides the ability to customize the proxy behavior. For instance, by creating a file named `.proxyrc.js` in project's root, we can manually redirect all API calls to a mock server listening on `localhost:8080`.

```js
const { createProxyMiddleware } = require('http-proxy-middleware')

module.exports = function (app) {
  const proxy = createProxyMiddleware('/api', {
    target: 'http://localhost:8080/',
  })
  app.use(proxy)
}
```

In order to serve API calls directly in Parcel's dev server, we just need to write our own middleware and wire it into the `connect` instance. Let's name it `mock-middleware`, and it has the following functions:

* Read source files from the `/mock` folder, and serve API calls with mock data.
* When the files are updated, refresh the APIs as well.

<!-- more -->

### Define mock files

```js
// /mock/user.js
const sendJson = require('send-data/json')

function login(req, res) {
  const { username, password } = req.body
  sendJson(req, res, {
    id: 1,
    nickname: 'Jerry',
  })
}

module.exports = {
  'POST /api/login': login,
}
```

Mock API are simple functions that accept standard Node.js request/response objects as parameters and receive and send data via them. The function is associated with a route string that will be used to match the incoming requests. To ease the processing of request and response data, we use [body-parser][5] to parse incoming JSON string into `req.body` object, and use [send-data][6] utility to send out JSON response, that helps setup the `Content-Type` header for us. Since `body-parser` is a middleware, we need to wire it into the `connect` app, before the `mock-middleware` we are about to implement.

```js
// /.proxyrc.js
const bodyParser = require('body-parser')
const { createMockMiddleware } = require('./mock-middleware')

module.exports = function (app) {
  app.use(bodyParser.json())
  app.use(createMockMiddleware('./mock')) // TODO
}
```

### Create router

To match the requests into different route functions, we use [route.js][7].

```js
// Create router and add rules.
const router = new Router()
router.addRoute('POST /api/login', login)

// Use it in a connect app middleware.
function middleware(req, res, next) {
  const { pathname } = url.parse(req.url)
  const m = router.match(req.method + ' ' + pathname)
  if (m) m.fn(req, res, m.param)
  else next()
}

app.use(middleware)
```

`route.js` supports parameters in URL path, but for query string we need to parse them on our own.

```js
module.exports = {
  // Access /user/get?id=1
  'GET /:controller/:action': (req, res, params) => {
    const { query } = url.parse(req.url)
    // Prints { controller: 'user', action: 'get' } { id: '1' }
    console.log(params, qs.parse(query))
    res.end()
  },
}
```

Combined with [glob][8], we scan files under `/mock` folder and add all of them to the router.

```js
glob.sync('./mock/**/*.js').forEach((file) => {
  const routes = require(path.resolve(file))
  Object.keys(routes).forEach((path) => {
    router.addRoute(path, routes[path])
  })
})
```

### Watch and reload

The next feature we need to implement is watch file changes under `/mock` folder and reload them. The popular [chokidar][9] package does the watch for us, and to tell Node.js reload these files, we simply clear the `require` cache.

```js
const watcher = chokidar.watch('./mock', { ignoreInitial: true })
const ptrn = new RegExp('[/\\\\]mock[/\\\\]')
watcher.on('all', () => {
  Object.keys(require.cache)
    .filter((id) => ptrn.test(id))
    .forEach((id) => {
      delete require.cache[id]
    })

  // Rebuild the router.
})
```

Now that we have all the pieces we need to create the `mock-middleware`, we wrap them into a class and provide a `createMockMiddleware` function for it. The structure is borrowed from [HttpProxyMiddleware][10]. Full code can be found on [GitHub][11].

```js
// /mock-middleware/index.js
class MockMiddleware {
  constructor(mockPath) {
    this.mockPath = mockPath
    this.createRouter()
    this.setupWatcher()
  }

  createRouter() {
    this.router = new Router()
    // ...
  }

  setupWatcher() {
    watcher.on('all', () => {
      // ...
      this.createRouter()
    })
  }

  middleware = (req, res, next) => {
    // ...
  }
}

function createMockMiddleware(mockPath) {
  const { middleware } = new MockMiddleware(mockPath)
  return middleware
}

module.exports = {
  createMockMiddleware,
}
```

## Standalone mock server

If you prefer setting up a dedicated server for mock API, either with [Express.js][12] or [JSON Server][13], it is easy to integrate with Parcel. Let's create a simple Express.js application first.

```js
// /mock-server/index.js
const express = require('express')

const app = express()
app.use(express.json())

app.post('/api/login', (req, res) => {
  const { username, password } = req.body
  res.json({
    id: 1,
    nickname: 'Jerry',
  })
})

const server = app.listen(8080, () => {
  console.log('Mock server listening on port ' + server.address().port)
})
```

To start the server while watching the file changes, use [nodemon][14].

```
yarn add -D nodemon
yarn nodemon --watch mock-server mock-server/index.js
```

Now configure Parcel to redirect API calls to the mock server.

```json
// /.proxyrc.json
{
  "/api": {
    "target": "http://127.0.0.1:8080/"
  }
}
```

Use [concurrently][15] to start up Parcel *and* mock server at the same time. In fact, it is more convenient to create a npm script for that. Add the following to `package.json`:

```json
{
  "scripts": {
    "start": "concurrently yarn:dev yarn:mock",
    "dev": "parcel",
    "mock": "nodemon --watch mock-server mock-server/index.js",
  }
}
```

## References

* https://codeburst.io/dont-use-nodemon-there-are-better-ways-fc016b50b45e
* https://github.com/Raynos/http-framework/wiki/Modules
* https://github.com/aaronblohowiak/routes.js#http-method-example
* https://mswjs.io/docs/comparison


[1]: https://parceljs.org/
[2]: https://parceljs.org/features/development/#api-proxy
[3]: https://github.com/senchalabs/connect
[4]: https://github.com/chimurai/http-proxy-middleware
[5]: https://github.com/expressjs/body-parser
[6]: https://github.com/Raynos/send-data
[7]: https://github.com/aaronblohowiak/routes.js
[8]: https://github.com/isaacs/node-glob
[9]: https://github.com/paulmillr/chokidar
[10]: https://github.com/chimurai/http-proxy-middleware/blob/v2.0.6/src/http-proxy-middleware.ts#L11
[11]: https://github.com/jizhang/blog-demo/tree/master/parcel-mock
[12]: https://github.com/expressjs/express
[13]: https://github.com/typicode/json-server
[14]: https://github.com/remy/nodemon
[15]: https://github.com/open-cli-tools/concurrently
