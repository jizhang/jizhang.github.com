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

Several notes on our alpha version. First, it plays well with the [Application Factories][3], that means the extension can be used to initialize with multiple application instances, with different configurations for web server, testing, etc. The key point is to provide an `init_app` method for different apps, and use the `current_app` proxy during work. To initialize the app:

```python
app = Flask(__name__)
db = SQLAlchemyAlpha(app)

# Or
db = SQLAlchemyAlpha()


def create_app() -> Flask:
    app = Flask(__name__)
    db.init_app(app)
    return app
```

Second, it plays well with [The Application Context][4], by storing data on current app context's stack, instead of on the extension instance, i.e. `self.some_attr`. When the `engine` attribute is first accessed, the extension creates a SQLAlchemy engine with the current app's config, and stores it on the current app context. When this context is popped, the engine object is also disposed, releasing the connection pool. Flask will automatically push and pop application context during request and command line interface. Here is an example of CLI:

```python
@app.cli.command()
def use_alpha() -> None:
    """Test alpha extension."""
    user_count = db_alpha.engine.execute('SELECT COUNT(*) FROM `user`').scalar_one()
    app.logger.info(f'User count: {user_count}')
```

## Keep engine around

The major problem of the alpha version is constantly creating and disposing SQLAlchemy engine objects. And we known [Engine][5] is rather a heavy object to construct, and should be kept around throughout the lifespan of the application. There is an extension point of the Flask app instance, where we can store data for the entire app.

```python
def init_app(self, app: Flask):
    url = app.config['SQLALCHEMY_DATABASE_URI']
    app.extensions['sqlalchemy'] = create_engine(url)
    app.teardown_appcontext(self.teardown)
```

For working with SQLAlchemy, we often prefer using sessions, so in the extension we need to create a session for each request, and properly close it after the context is popped.

```python
def connect(self) -> Session:
    return Session(current_app.extensions['sqlalchemy'])

def teardown(self, exception) -> None:
    ctx = _app_ctx_stack.top
    if hasattr(ctx, 'sqlalchemy_session'):
        ctx.sqlalchemy_session.close()

@property
def session(self) -> Session:
    ctx = _app_ctx_stack.top
    if ctx is None:
        raise RuntimeError(_app_ctx_err_msg)
    if not hasattr(ctx, 'sqlalchemy_session'):
        ctx.sqlalchemy_session = self.connect()
    return ctx.sqlalchemy_session
```

To use session in a view:

```python
@app.get('/api/user/list')
def get_user_list() -> Response:
    users: List[User] = db.session.query(User).all()
    return jsonify(users=users)
```

You may wonder why we don't use [`scoped_session`][6], which is the recommended way to use sessions in a multi-thread environment. The answer is simple: an application context will not be shared by different workers, so it is safe to use the same session throughout the request. And, since session is a light-weight object, it is OK to create it on every request. Check Werkzeug [Context Locals][7] for more information.

## Define models in a native way



## Appendix I: Serialize SQLAlchmey models to JSON

## Appendix II: Support multiple database binds


[1]: https://flask-sqlalchemy.palletsprojects.com/
[2]: https://flask.palletsprojects.com/en/2.1.x/extensiondev/
[3]: https://flask.palletsprojects.com/en/2.1.x/patterns/appfactories/
[4]: https://flask.palletsprojects.com/en/2.1.x/appcontext/
[5]: https://docs.sqlalchemy.org/en/14/core/connections.html
[6]: https://docs.sqlalchemy.org/en/14/orm/contextual.html
[7]: https://werkzeug.palletsprojects.com/en/2.1.x/local/
