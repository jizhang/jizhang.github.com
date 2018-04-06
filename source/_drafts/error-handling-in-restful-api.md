---
title: Error Handling in RESTful API
tags: [javascript, python, frontend]
categories: Programming
---

![RESTful API](/images/restful-api.png)

RESTful API is a common tool of building web services, especially in front and backend separated application. It is based on HTTP protocol, which is simple, text-oriented, and well supported by various languages, browsers or clients. However, REST is not yet standardized, so that the developers need to decide how to design their APIs. One of the decisions is error handling. Should I use HTTP status code? How to handle form validation errors, etc. This article will propose an error handling mechanism for RESTful API, based on my daily work and understanding of this technique.

## Types of Errors

I tend to categorize errors into two type, global and local. Global errors include requesting an unkown API url,not being authorized to access this API, or there's something wrong with the server code, unexcepted and fatal. These errors should be caught by the web framework, no customized handling in individual API function.

Local errors, on the other hand, are closely related to the current API. Examples are form validation, violation of unique constraint, or other expected errors. We need to write specific codes to catch these errors, and raise a global error with message and payload for framework to catch and respond with.

Flask, for instance, provides a mechanism to catch global exception:

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

```text
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
    "status": 40001,
    "message": "username cannot be empty",
    "ext": 1
}
```

### Use HTTP Status Code

200
400
401
403
404
500

### Global and Local Error


### More Error Information

```javascript
{
    "status": 40001,
    "message": "form validation failed"
    "errors": [
        { "name": "username", "error": "username cannot be empty" },
        { "name": "password", "error": "password minimum length 6" }
    ]
}
```

## Fetch API

## MobX Flow Example

## References

* https://en.wikipedia.org/wiki/Representational_state_transfer
