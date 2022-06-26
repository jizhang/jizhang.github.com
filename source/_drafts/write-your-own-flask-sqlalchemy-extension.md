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

You may wonder why we don't use [`scoped_session`][6], which is the recommended way to use sessions in a multi-thread environment. The answer is simple: an application context will not be shared by different workers, so it is safe to use the same session throughout the request. And, since session is a light-weight object, it is OK to create it on every request. Check Werkzeug [Context Locals][7] for more information.

## Define models in a native way

Now that we have a simple but fully functional flask-sqlalchemy extension, we can start writing models in a native way.

```python
from sqlalchemy import Column, Integer, String
from sqlalchemy.orm import declarative_base

Base = declarative_base()


class User(Base):
    __tablename__ = 'user'
    id = Column(Integer, primary_key=True)
    username = Column(String)
```

There is not `db.Model` or `db.Integer`. The model base class need to be declared explicitly, as well as the table name of each model. Add a CLI that creates the tables:

```python
@app.cli.command()
def init_db() -> None:
    """Initialize database."""
    Base.metadata.create_all(db.session.get_bind())
```

And execute query in a view:

```python
@app.get('/api/user/list')
def get_user_list() -> Response:
    users: List[User] = db.session.query(User).all()
    return jsonify(users=users)
```

To enable type hints for SQLAlchemy models, install `sqlalchemy2-stubs` and enable the plugin in `mypy.ini`:

```ini
[mypy]
warn_unused_configs = True
plugins = sqlalchemy.ext.mypy.plugin
```

Now `user.id` will have the type `Column[Integer]`. This will continue to work in SQLAlchemy 2.x, except no extra dependency is needed. You may want to read the document [Mypy Support for ORM Mappings][8].

Source code in this article can be found on [GitHub][9].

## Appendix I: Serialize SQLAlchemy models to JSON

Flask's built-in JSON serializer does not recoganize SQLAlchemy models, neither the frequently used `Decimal` and `datetime` objects. But we can easily enhance it with a custom encoder:

```python
from decimal import Decimal
from datetime import datetime

from flask.json import JSONEncoder
from sqlalchemy.engine.row import Row
from sqlalchemy.orm.decl_api import DeclarativeMeta


class CustomEncoder(JSONEncoder):
    def default(self, obj):
        if isinstance(obj, Decimal):
            return float(obj)

        if isinstance(obj, datetime):
            return obj.strftime('%Y-%m-%d %H:%M:%S')

        if isinstance(obj, Row):
            return dict(obj)

        if isinstance(obj.__class__, DeclarativeMeta):
            return {c.name: getattr(obj, c.name) for c in obj.__table__.columns}

        return super().default(obj)
```

`Row` is for core engine use case, and `DeclarativeMeta` for ORM. Add a line when creating the app:

```python
app.json_encoder = CustomEncoder
```

## Appendix II: Support multiple database binds

If you are interested in supporting multiple binds like Flask-SQLAlchmey does, here is a proof of concept. But for such complex scenario, I suggest use the opensource extension instead, for it is more mature, feature-complete, and fully tested.

This time we do not create engine on app startup. We create scoped sessions on demand.

```python
def connect(self, name: str) -> scoped_session:
    if name == 'default':
        url = current_app.config['SQLALCHEMY_DATABASE_URI']
    else:
        url = current_app.config['SQLALCHEMY_BINDS'][name]

    engine = create_engine(url, echo=echo)
    session_factory = sessionmaker(bind=engine)
    return scoped_session(session_factory)
```

The configuration style mimics Flask-SQLAlchemy. This version of `connect` will return a properly configured `scoped_session` object, and it will be shared among different workers, so we store it in the app's `extensions` dict.

```python
class Holder:
    sessions: Dict[str, scoped_session]
    lock: Lock

    def __init__(self):
        self.sessions = {}
        self.lock = Lock()


class SQLAlchemyMulti:
    def init_app(self, app: Flask):
        app.extensions['sqlalchemy_multi'] = Holder()
        app.teardown_appcontext(self.teardown)

    def get_session(self, name: str = 'default') -> Session:
        holder: Holder = current_app.extensions['sqlalchemy_multi']
        with holder.lock:
            if name not in holder.sessions:
                holder.sessions[name] = self.connect(name)
            return holder.sessions[name]()
```

Note the creation of the `scoped_session` object is not thread-safe, so we guard it with a lock. Again, this lock should not be stored as extension instance's attribute, we create a `Holder` class to hold both the lock and scoped sessions.

Do not forget to do the cleanup work:

```python
def teardown(self, exception) -> None:
    holder: Holder = current_app.extensions['sqlalchemy_multi']
    for session in holder.sessions.values():
        session.remove()
```

`scoped_session.remove` will invoke `close` on the session and remove it from its registry. Next request will get a brand new session object.

We can verify that it uses the desired connection:

```python
@app.cli.command()
def product_db() -> None:
    """Access product database."""
    db_file = db_multi.get_session('product_db').execute(
        text("""
        SELECT `file` FROM pragma_database_list
        WHERE `name` = :name
        """),
        {
            'name': 'main',
        }
    ).scalar()
    app.logger.info(f'Database file: {db_file}')
```


[1]: https://flask-sqlalchemy.palletsprojects.com/
[2]: https://flask.palletsprojects.com/en/2.1.x/extensiondev/
[3]: https://flask.palletsprojects.com/en/2.1.x/patterns/appfactories/
[4]: https://flask.palletsprojects.com/en/2.1.x/appcontext/
[5]: https://docs.sqlalchemy.org/en/14/core/connections.html
[6]: https://docs.sqlalchemy.org/en/14/orm/contextual.html
[7]: https://werkzeug.palletsprojects.com/en/2.1.x/local/
[8]: https://docs.sqlalchemy.org/en/14/orm/extensions/mypy.html
[9]: https://github.com/jizhang/blog-demo/tree/master/native-sqlalchemy
