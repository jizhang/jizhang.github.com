---
title: Configure Logging for Flask SQLAlchemy Project
tags:
  - flask
  - sqlalchemy
  - python
categories:
  - Programming
date: 2022-08-10 10:56:07
---


In Python, the built-in [`logging`][1] module is the standard way of doing application logs, and most third-party libraries integrate well with it. For instance, [Flask][2] creates a default `app.logger` with a `StreamHandler` that writes to standard error. [SQLAlchemy][3] uses a logger named `sqlalchemy` and allow us to further customize its behaviour. This article shows how to configure logging for Flask and SQLAlchemy, both in debug mode and production mode.

## Default logging behaviour of Flask

According to Flask document, when the `app.logger` property is accessed for the first time, it creates a logger with the name of the application, usually the module name you used in `app = Flask(__name__)`. The logging level is set to `DEBUG` if current application is in debug mode, or `NOTSET` and lets parent loggers decide the level. Then it checks if a log handler has already been added to the logger or any parent loggers, otherwise it adds a default one. The log format is as follows:

```python
logging.Formatter("[%(asctime)s] %(levelname)s in %(module)s: %(message)s")
```

In application, we can invoke the logging methods on `app.logger`:

```python
from modern import app  # modern is the project name

# Invoke in modern.views.user module.
app.logger.info('Get user list')
```

The output is:

```
[2022-08-08 18:33:11,451] INFO in user: Get user list
```

In production, the root logger is set to `WARNING` level by default, so only warning, error, and critical messages will be logged.

<!-- more -->

## Customize application logging

There are several things we can improve in application logging:

* Use the full module name as the logger name, and print it in logs like `INFO in modern.views.user`. This also allows us to configure logging for parent modules.
* Change the log level to INFO in production, so that we may print some useful information for debugging.
* Simplify the use of logger when applying the Flask [Application Factories][4] pattern.

In order to give the logger the full module name, we need to create it on our own. Then configuring level will be very easy.

```python
def create_app() -> Flask:
    app = Flask(__name__)
    configure_logging(app)
    return app


def configure_logging(app: Flask):
    logging.basicConfig(format='[%(asctime)s] %(levelname)s %(name)s: %(message)s')
    logging.getLogger().setLevel(logging.INFO)

    if app.debug:
        logging.getLogger().setLevel(logging.DEBUG)
```

Here we use the Flask app factory pattern, and configure logging right after we create the app instance. This is necessary because once `app.logger` is accessed, default behaviour will be set up. The log format is similar to the default, except we use `%(name)s` instead of `%(module)s`. Then we set the root logger level to `INFO`, and if we are in debug mode, `DEBUG` level is used. Besides, `basicConfig` adds a default handler that logs into standard error. This is sufficient for applications running in containers.

To use logger in modules:

```python
import logging

from flask import current_app, jsonify, Response

logger = logging.getLogger(__name__)


@current_app.get('/api/user/list')
def get_user_list() -> Response:
    logger.info('Get user list in view.')
    return jsonify(users=[])
```

Output:

```
[2022-08-09 18:12:12,420] INFO modern.views.user: Get user list in view.
```

With app factory pattern, we need to replace `app.logger` with `current_app.logger`, and it is a little bit verbose. Dedicated logger for each module sovles this problem, too.

### Fix Werkzeug logging

In debug mode, [Werkzeug][5] will output access logs twice:

```
[2022-08-09 18:17:28,530] INFO werkzeug:  * Restarting with stat
 * Debugger is active!

127.0.0.1 - - [09/Aug/2022 18:17:30] "GET /api/user/list HTTP/1.1" 200 -
[2022-08-09 18:17:30,355] INFO werkzeug: 127.0.0.1 - - [09/Aug/2022 18:17:30] "GET /api/user/list HTTP/1.1" 200 -
```

To fix it, we remove the extra handler under `werkzeug` logger:

```python
if app.debug:
    # Fix werkzeug handler in debug mode
    logging.getLogger('werkzeug').handlers = []
```

In production mode, the access log is controlled by WSGI server:

```
gunicorn -b 127.0.0.1:5000 --access-logfile - 'modern:create_app()'
```

## Log SQLAlchemy queries in debug mode

To log queries, SQLAlchemy gives us two options: create engine with `echo=True`, or configure the logger ourselves. Only use one approach or you will get duplicate logs. For [Flask-SQLAlchemy][6] users, use the following config:

```python
SQLALCHEMY_ECHO = True
```

I prefer using the standard logging module. All SQLAlchemy loggers are under the name `sqlalchemy`, and they are by default in `WARNING` level. To enable query logs, change the level of `sqlalchemy.engine` logger to `INFO`. If you also want to get the query result printed, set to `DEBUG`.

```python
if app.debug:
    # Make sure engine.echo is set to False
    logging.getLogger('sqlalchemy.engine').setLevel(logging.INFO)
```

Output:

```
[2022-08-10 10:41:57,089] INFO sqlalchemy.engine.Engine: BEGIN (implicit)
[2022-08-10 10:41:57,090] INFO sqlalchemy.engine.Engine: SELECT user.id AS user_id, user.username AS user_username FROM user
[2022-08-10 10:41:57,090] INFO sqlalchemy.engine.Engine: [generated in 0.00015s] {}
[2022-08-10 10:41:57,091] INFO sqlalchemy.engine.Engine: ROLLBACK
```

## Conclusion

Flask's built-in `app.logger` is easy to use. But instead, we create our own loggers to fine-tune the configs, with Python standard logging module. It is also true for SQLAlchemy logs. The loggers are well defined in `sqlalchemy.engine`, `sqlalchemy.orm`, etc., so that we can change the configs easily. Demo code can be found on [GitHub][7].


[1]: https://docs.python.org/3/library/logging.html
[2]: https://flask.palletsprojects.com/en/2.1.x/logging/
[3]: https://docs.sqlalchemy.org/en/14/core/engines.html#configuring-logging
[4]: https://flask.palletsprojects.com/en/2.1.x/patterns/appfactories/
[5]: https://werkzeug.palletsprojects.com/
[6]: https://flask-sqlalchemy.palletsprojects.com/en/2.x/config/
[7]: https://github.com/jizhang/blog-demo/tree/master/modern-flask
