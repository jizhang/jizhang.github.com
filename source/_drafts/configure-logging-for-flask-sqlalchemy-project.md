---
title: Configure Logging for Flask SQLAlchemy Project
categories: [Programming]
tags: [flask, sqlalchemy, python]
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


[1]: https://docs.python.org/3/library/logging.html
[2]: https://flask.palletsprojects.com/en/2.1.x/logging/
[3]: https://docs.sqlalchemy.org/en/14/core/engines.html#configuring-logging
