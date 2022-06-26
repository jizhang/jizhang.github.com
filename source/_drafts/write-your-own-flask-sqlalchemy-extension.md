---
title: Write Your Own Flask SQLAlchemy Extension
categories: Programming
tags: [python, flask, sqlalchemy]
---

When it comes to connecting to database in Flask project, we tend to use the [Flask-SQLAlchemy][1] extension that handles the lifecycle of database connection, add a certain of utilities for defining models and executing queries, and integrate well with the Flask framework. However, if you are developing a rather simple project with Flask and SQLAlchemy, and do not want to depend on another third-party library, or you prefer using SQLAlchemy directly, making the model layer agnostic of web frameworks, you can write your own extension. Besides, you will gain better type hints for SQLAlchemy model, and possibly easier migration to SQLAlchemy 2.x. This article will show you how to integrate SQLAlchemy 1.4 with Flask 2.1.

## The alpha version

In the official document [Flask Extension Development][2], it shows us writing a sqlite3 extension that plays well with Flask application context. So our first try is to replace sqlite3 with SQLAlchemy:

```python
from typing import Optional

from flask import Flask, current_app
from flask.globals import _app_ctx_stack, _app_ctx_err_msg
from sqlalchemy import create_engine
from sqlalchemy.engine import Engine


class SQLAlchemyAlpha:
    def __init__(self, app: Optional[Flask] = None):
        self.app = app
        if app is not None:
            self.init_app(app)

    def init_app(self, app: Flask):
        app.config.setdefault('SQLALCHEMY_DATABASE_URI', 'sqlite://')
        app.teardown_appcontext(self.teardown)

    def connect(self) -> Engine:
        return create_engine(current_app.config['SQLALCHEMY_DATABASE_URI'])

    def teardown(self, exception) -> None:
        ctx = _app_ctx_stack.top
        if hasattr(ctx, 'sqlalchemy'):
            ctx.sqlalchemy.dispose()

    @property
    def engine(self) -> Engine:
        ctx = _app_ctx_stack.top
        if ctx is not None:
            if not hasattr(ctx, 'sqlalchemy'):
                ctx.sqlalchemy = self.connect()
            return ctx.sqlalchemy
        raise RuntimeError(_app_ctx_err_msg)
```

<!-- more -->


[1]: https://flask-sqlalchemy.palletsprojects.com/
[2]: https://flask.palletsprojects.com/en/2.1.x/extensiondev/
