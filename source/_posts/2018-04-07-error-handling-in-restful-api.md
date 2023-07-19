---
title: Error Handling in RESTful API
tags:
  - javascript
  - python
  - frontend
  - restful
categories: Programming
date: 2018-04-07 14:49:19
---


![RESTful API](/images/restful-api.png)

RESTful API is a common tool of building web services, especially in front and back-end separated application. It is based on HTTP protocol, which is simple, text-oriented, and well supported by various languages, browsers or clients. However, REST is not yet standardized, so that the developers need to decide how to design their APIs. One of the decisions is error handling. Should I use HTTP status code? How to handle form validation errors, etc. This article will propose an error handling mechanism for RESTful API, based on my daily work and understanding of this technique.

## Types of Errors

I tend to categorize errors into two types, global and local. Global errors include requesting an unknown API url, not being authorized to access this API, or there's something wrong with the server code, unexpected and fatal. These errors should be caught by the web framework, no customized handling in individual API function.

Local errors, on the other hand, are closely related to the current API. Examples are form validation, violation of unique constraint, or other expected errors. We need to write specific codes to catch these errors, and raise a global error with message and payload for framework to catch and respond with.

Flask, for instance, provides a mechanism to catch exceptions globally:

```python
class BadRequest(Exception):
    """Custom exception class to be thrown when local error occurs."""
    def __init__(self, message, status=400, payload=None):
        self.message = message
        self.status = status
        self.payload = payload


@app.errorhandler(BadRequest)
def handle_bad_request(error):
    """Catch BadRequest exception globally, serialize into JSON, and respond with 400."""
    payload = dict(error.payload or ())
    payload['status'] = error.status
    payload['message'] = error.message
    return jsonify(payload), 400


@app.route('/person', methods=['POST'])
def person_post():
    """Create a new person object and return its ID"""
    if not request.form.get('username'):
        raise BadRequest('username cannot be empty', 40001, { 'ext': 1 })
    return jsonify(last_insert_id=1)
```

<!-- more -->

## Error Response Payload

When you post to `/person` with an empty `username`, it'll return the following error response:

```text
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "status": 40001,
  "message": "username cannot be empty",
  "ext": 1
}
```

There're several parts in this response: HTTP status code, a custom status code, error message, and some extra information.

### Use HTTP Status Code

HTTP status code itself provides rich semantics for errors. Generally `4xx` for client-side error and `5xx` server-side. Here's a brief list of commonly used codes:

* `200` Response is OK.
* `400` Bad request, e.g. user posts some in valid data.
* `401` Unauthorized. With `Flask-Login`, you can decorate a route with `@login_required`, and if the user hasn't logged in, `401` will be returned, and client-side can redirect to login page.
* `403` Access is forbidden.
* `404` Resource not found.
* `500` Internal server error. Usually for unexpected and irrecoverable exceptions on the server-side.

### Custom Error Code

When client receives an error, we can either open a global modal dialog to show the message, or handle the errors locally, such as displaying error messages below each form control. For this to work, we need to give these local errors a special coding convention, say `400` for global error, while `40001` and `40002` will trigger different error handlers.

```javascript
fetch().then(response => {
  if (response.status == 400) { // http status code
    response.json().then(responseJson => {
      if (responseJson.status == 400) { // custom error code
        // global error handler
      } else if (responseJson.status == 40001) { // custom error code
        // custom error handler
      }
    })
  }
})
```

### More Error Information

Sometimes it is ideal to return all validation errors in one response, and we can use `payload` to achieve that.

```javascript
{
  "status": 40001,
  "message": "form validation failed"
  "errors": [
    { "name": "username", "error": "username cannot be empty" },
    { "name": "password", "error": "password minimum length is 6" }
  ]
}
```

## Fetch API

For AJAX request, [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) becomes the standard library. We can wrap it into a function that does proper error handling. Full code can be found in GitHub ([link](https://github.com/jizhang/blog-demo/blob/master/rest-error/src/request.js)).

```javascript
function request(url, args, form) {
  return fetch(url, config)
    .then(response => {
      if (response.ok) {
        return response.json()
      }

      if (response.status === 400) {
        return response.json()
          .then(responseJson => {
            if (responseJson.status === 400) {
              alert(responseJson.message) // global error handler
            }
            // let subsequent "catch()" in the Promise chain handle the error
            throw responseJson
          }, error => {
            throw new RequestError(400)
          })
      }

      // handle predefined HTTP status code respectively
      switch (response.status) {
        case 401:
          break // redirect to login page
        default:
          alert('HTTP Status Code ' + response.status)
      }

      throw new RequestError(response.status)
    }, error => {
      alert(error.message)
      throw new RequestError(0, error.message)
    })
}
```

This method will reject the promise whenever an error happens. Invokers can catch the error and check its `status`. Here's an example of combining this approach with MobX and ReactJS:

```javascript
// MobX Store
loginUser = flow(function* loginUser(form) {
  this.loading = true
  try {
    // yield may throw an error, i.e. reject this Promise
    this.userId = yield request('/login', null, form)
  } finally {
    this.loading = false
  }
})

// React Component
login = () => {
  userStore.loginUser(this.state.form)
    .catch(error => {
      if (error.status === 40001) {
        // custom error handler
      }
    })
}
```

## References

* https://en.wikipedia.org/wiki/Representational_state_transfer
* https://alidg.me/blog/2016/9/24/rest-api-error-handling
* https://www.wptutor.io/web/js/generators-coroutines-async-javascript
