---
title: Define API Data Models with Pydantic
categories: [Programming]
tags: [python, pydantic, flask, openapi]
---

In modern architecutre, frontend and backend are separated and maintained by different teams. To cooperate, backend exposes services as API endpoints with carefully designed data models, for both request and response. In Python, there are numerous ways to complete this task, such as WTForms, marshmallow. There are also frameworks that are designed to build API server, like FastAPI, Connexion, both are built around OpenAPI specification. In this artile, I will introduce [Pydantic][1], a validation and serialization library for Python, to build and enforce API request and response models. The web framework I choose is Flask, but Pydantic is framework agnostic and can also be used in non-web applications.

![Pydantic](/images/pydantic.png)

## Define response model

After `pip install pydantic`, let's define a simple response model to return the currently logged-in user:

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    username: str
    last_login: datetime

@app.get('/current-user')
def current_user() -> dict:
    user = User(id=1, username='jizhang', last_login=datetime.now())
    return user.model_dump(mode='json')
```

Then use httpie to test the API:

```
% http localhost:5000/current-user
HTTP/1.1 200 OK
Content-Type: application/json

{
    "id": 1,
    "username": "jizhang",
    "last_login": "2024-01-25T10:25:23.670431"
}
```

* We create Pydantic model by extending `BaseModel`, which is the basic approach. There are others ways like `dataclass`, `TypeAdapter`, or dynamic creation of models.
* Model fields are simply defined by class attributes and type annotations. Unlike other SerDe libraries, Pydantic is natively built with Python type hints. If you are not familiar with it, please check out my previous [blog post][2].
* In the API, we manually create a model instance `user`. Usually we create them from request body or database models, which will be demonstrated later.
* Then we serialize, or "dump" the model into a Python dict, that in turn is transformed by Flask into a JSON string. We can also use `user.model_dump_json()`, which returns the JSON string directly, but then the response header needs to be manually set to `application/json`, so we would rather let Flask do the job.
* `mode="json"` tells Pydantic to serialize field values into JSON representable types. For instance, `datetime` and `Decimal` will be converted to string. Flask can also do this conversion, but we prefer keeping serialization in Pydantic model for clarity and ease of change.

<!-- more -->

### Create from SQLAlchemy model

Using model contructor to create instance is one way. We can also create from a Python dictionary:

```python
user = User.model_validate({'id': 1, 'username': 'jizhang', 'last_login': datetime.now()})
```

Or an arbitrary class instance:

```python
class UserDto:
    def __init__(self, id: int, username: str, last_login: datetime):
        self.id = id
        self.username = username
        self.last_login = last_login

user = User.model_validate(UserDto(1, 'jizhang', datetime.now()), from_attributes=True)
```

`UserDTO` can also be a Python dataclass. You may notice the `from_attributes` parameter, which means field values are extracted from object's attributes, instead of dictionary key value pairs. If the model is always created from objects, we can add this configuration to the model:

```python
class User(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    ...

user = User.model_validate(user_dto)
```

This is actually how we integrate with SQLAlchemy, creating Pydantic model instance from SQLAlchemy model instance:

```python
from sqlalchemy.orm import Mapped, mapped_column

class UserOrm(Base):
    __tablename__ = 'user'

    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str]
    last_login: Mapped[datetime]

user_orm = db.session.get_one(UserOrm, 1)
user = User.model_validate(user_orm)
```

SQLModel

* Define response model
    * Installation, mypy plugin
    * Python typing
    * From sqlalchemy, sqlmodel
    * Custom serializer, e.g. datetime
    * Computed fields
    * Alias, snake_case to camelCase
    * Nested
    * Exclude
    * Context: excluded field, model_validate context param
* Define request model
    * Modeling query string
    * Custom deserializer, return a different object like @post_load
    * Default value, default factory
    * Required fields
    * Alias
    * Type conversion, datetime
* Validation
    * Validate route variables
    * String, number, decimal
    * Choices: enum, literal
    * Annotation, Field, Gt
    * Pydantic types
    * Custom validator
    * Validation error
* OpenAPI
    * JSON Schema, example data
    * Manually reference
    * openapi-pydantic, spectree, fastapi

## References
* https://docs.pydantic.dev/latest/concepts/models/


[1]: https://pydantic.dev/
[2]: https://shzhangji.com/blog/2024/01/19/python-static-type-check/
