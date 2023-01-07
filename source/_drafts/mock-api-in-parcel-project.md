---
title: Mock API in Parcel Project
categories: Programming
tags: [frontend, parcel, connect]
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

* Read source files from the `./mock` folder, and serve API calls with mock data.
* When the files are updated, refresh the APIs as well.

<!-- more -->

### Define mock files

### Create router

### Watch and reload

## Other options
* express + nodemon + concurrently
* json-server, mockjs
* fetch-mock, axios-mock-adapter, msw

## References
* https://codeburst.io/dont-use-nodemon-there-are-better-ways-fc016b50b45e
* https://github.com/chimurai/http-proxy-middleware/blob/master/src/http-proxy-middleware.ts
* https://github.com/Raynos/http-framework/wiki/Modules


[1]: https://parceljs.org/
[2]: https://parceljs.org/features/development/#api-proxy
[3]: https://github.com/senchalabs/connect
[4]: https://github.com/chimurai/http-proxy-middleware
