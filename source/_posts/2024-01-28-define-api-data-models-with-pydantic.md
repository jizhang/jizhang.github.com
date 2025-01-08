---
title: Define API Data Models with Pydantic
tags:
  - python
  - pydantic
  - flask
  - openapi
categories:
  - Programming
date: 2024-01-28 17:53:28
---


In modern architecture, frontend and backend are separated and maintained by different teams. To cooperate, backend exposes services as API endpoints with carefully designed data models, for both request and response. In Python, there are numerous ways to complete this task, such as WTForms, marshmallow. There are also frameworks that are designed to build API server, like FastAPI, Connexion, both are built around OpenAPI specification. In this article, I will introduce [Pydantic][1], a validation and serialization library for Python, to build and enforce API request and response models. The web framework I choose is Flask, but Pydantic is framework-agnostic and can also be used in non-web applications.

![Pydantic](/images/api-pydantic/pydantic.png)


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

Then use [httpie][9] to test the API:

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

Using model constructor to create instance is one way. We can also create from a Python dictionary:

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

`UserDto` can also be a Python dataclass. You may notice the `from_attributes` parameter, which means field values are extracted from object's attributes, instead of dictionary key value pairs. If the model is always created from objects, we can add this configuration to the model:

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
    password: Mapped[str]
    last_login: Mapped[datetime]

user_orm = db.session.get_one(UserOrm, 1)
user = User.model_validate(user_orm)
```

Would it be nice if our model is both Pydantic model *and* SQLAlchemy model? [SQLModel][3] is exactly designed for this purpose:

```python
from sqlmodel import SQLModel, Field

class User(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    username: str
    last_login: datetime

# Read from database
user = db.session.get_one(User, 1)
user.model_dump(mode='json')

# Create from request data
user = User.model_validate(request.get_json())
db.session.add(user)
```

But personally I am not in favor of this approach, for it mixes classes from two layers, domain layer and presentation layer. Now the class has two reasons to change, thus violating the single responsibility principle. Use it judiciously.


### Nested models

To return a list of users, we can either create a dedicated response model:

```python
class UserListResponse(BaseModel):
    users: list[User]

user_orms = user_svc.get_list()
response = UserListResponse(users=user_orms)
response.model_dump(mode='json')
```

Or, if you prefer to return a list, we can create a custom type with `TypeAdapter`:

```python
from pydantic import TypeAdapter

UserList = TypeAdapter(list[User])

user_orms = user_svc.get_list()
user_list = UserList.validate_python(user_orms)
UserList.dump_python(user_list, mode='json')
```

I recommend the first approach, since it would be easier to add model attributes in the future.


### Custom serialization

By default, `datetime` object is serialized into ISO 8601 string. If you prefer a different representation, custom serializers can be added. There are several ways to accomplish this task. Decorate a class method with `field_serializer`:

```python
from pydantic import field_serializer

class User(BaseModel):
    last_login: datetime

    @field_serializer('last_login')
    def serialize_last_login(self, value: datetime) -> str:
        return value.strftime('%Y-%m-%d %H:%M:%S')
```

Create a new type with custom serializer:

```python
from typing import Annotated
from pydantic import PlainSerializer

def format_datetime(dt: datetime):
    return dt.strftime('%Y-%m-%d %H:%M:%S')

CustomDatetime = Annotated[datetime, PlainSerializer(format_datetime)]

class User(BaseModel):
    last_login: CustomDatetime
```

`Annotated` is widely used in Pydantic, to attach extra information like custom serialization and validation to an existing type. In this example, we use a `PlainSerializer`, which takes a function or lambda to serialize the field. There is also a `WrapSerializer`, that can be used to apply transformation before and after the default serializer.

Finally, there is the `model_serializer` decorator that can be used to transform the whole model, as well as individual fields.

```python
from pydantic import model_serializer

class User(BaseModel):
    id: int
    last_login: datetime

    @model_serializer
    def serialize_model(self) -> dict[str, Any]:
        return {
            'id': self.id,
            'last_login': self.last_login.strftime('%Y-%m-%d %H:%M:%S'),
        }

class UserListResponse(BaseModel):
    users: list[User]

    @model_serializer
    def serialize_model(self) -> list:
        return self.users
```

Now `UserListResponse` will be dumped into a list, instead of a dictionary.


### Field alias

Sometimes we want to change the key name in serialized data. For instance, change `users` to `userList`:

```python
from pydantic import Field

class UserListResponse(BaseModel):
    users: list[User] = Field(serialization_alias='userList')

response = UserListResponse(users=user_orms)
response.model_dump(mode='json', by_alias=True)
# {"userList": []}
```

`serialization_alias` indicates that the alias is only used for serialization. When creating models, we still use `users` as the key. To change both keys to `userList`, use `Field(alias='userList')`. If this conversion is universal, say you want all your request and response data to use camelCase for keys, add these configurations to your model:

```python
from pydantic.alias_generators import to_camel

class UserListResponse(BaseModel):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)

    user_list: list[User]

# Create from request data
UserListResponse.model_validate({'userList': []})
# This still works
UserListResponse(user_list=[])

# Dump to response data
response.model_dump(mode='json', by_alias=True)
# {"userList": []}
```


### Computed fields

Fields may derive from other fields:

```python
from flask import url_for
from pydantic import computed_field

class Article(BaseModel):
    id: int
    title: str

    @computed_field  # type: ignore[misc]
    @property
    def link(self) -> str:
        return url_for('article', id=self.id)
```

If the field requires extra information, we can add a private attribute to the model. The attribute's name starts with an underscore, and Pydantic will ignore it in serialization and validation.

```python
from pydantic import PrivateAttr

class Article(BaseModel):
    category_id: int
    _categories: dict[int, str] = PrivateAttr(default_factory=dict)

    @computed_field  # type: ignore[misc]
    @property
    def category_name(self) -> str:
        return self._categories.get(self.category_id, 'N/A')

article = Article(category_id=1)
article._categories = {1: 'Big Data'}
article.model_dump(mode='json')
# {"category_id": 1, "category_name": "Big Data"}
```


## Define request model

```python
from pydantic import Field

class UserForm(BaseModel):
    username: str
    password: str = Field(exclude=True)

class CreateUserResponse(BaseModel):
    id: int

@app.post('/create-user')
def create_user() -> dict:
    form = UserForm.model_validate(request.get_json())
    user_orm = UserOrm(**dict(form), last_login=datetime.now())
    db.session.add(user_orm)
    db.session.commit()

    response = CreateUserResponse(id=user_orm.id)
    return response.model_dump(mode='json')
```

* `model_validate` takes the dictionary returned by `get_json`, validates it, and constructs a model instance. There is also a `model_validate_json` method that accepts JSON string.
* The validated form data is then passed to an ORM model. Usually this is done by manual assignments, because fields like `password` need to be properly encrypted.
* `Field(exclude=True)` indicates that this field will be excluded in serialization. This is helpful when you do not want some information leaking to the client.

```
% http localhost:5000/create-user username=jizhang password=password
HTTP/1.1 200 OK
Content-Type: application/json

{
    "id": 3
}
```

Query parameters can be modeled in a similar way:

```python
class SearchForm(BaseModel):
    tags: str
    keyword: str

@app.get('/article/search')
def article_search() -> dict:
    form = SearchForm.model_validate(request.args.to_dict())
    return form.model_dump(mode='json')
```

Use `==` to tell httpie to use GET method:

```
% http localhost:5000/article/search tags==a,b,c keyword==test
HTTP/1.1 200 OK
Content-Type: application/json

{
    "keyword": "test",
    "tags": "a,b,c"
}
```


### Custom deserialization

Let's see how to parse `tags` string to a list of tags:

```python
class SearchForm(BaseModel):
    tags: list[str]
    keyword: str

    @field_validator('tags', mode='before')
    @classmethod
    def validate_tags(cls, value: str) -> list[str]:
        return value.split(',')

form = SearchForm.model_validate(request.args.to_dict())
print(form.tags)
# ['a', 'b', 'c']
```

`field_validator` is used to compose custom validation rules, which will be discussed in a later section. Normally it executes after Pydantic has done the default validation. In this case, `tags` is declared as `list[str]` and Pydantic would raise an error when a string is passed to it. So we use `mode='before'` to apply this function on the raw input data, and transform it into a list of tags.

There are also annotated validator and `model_validator`:

```python
from pydantic import BeforeValidator

def parse_tags(tags: str) -> list[str]:
    return tags.split(',')

TagList = Annotated[list[str], BeforeValidator(parse_tags)]

class SearchForm(BaseModel):
    tags: TagList
    keyword: Optional[str]

    @model_validator(mode='before')
    @classmethod
    def validate_model(cls, data: dict[str, Any]) -> dict[str, Any]:
        if not data['keyword'].strip():
            data['keyword'] = None
        return data
```


### Required field and default value

By default, all model attributes are required. Though `keyword` is defined as `Optional`, Pydantic will still raise an error if `keyword` is missing in the input data.

```python
SearchForm.model_validate_json('{"tags": "a,b,c"}')
# ValidationError keyword Field required

form = SearchForm.model_validate_json('{"tags": "a,b,c", "keyword": null}')
print(form.keyword)
# None
```

There are several ways to provide a default value for missing keys:

```python
from typing import Optional, Annotated
from datetime import datetime
from uuid import uuid4
from pydantic import Field

class SearchForm(BaseModel):
    keyword_1: Optional[str] = None
    keyword_2: Optional[str] = Field(default=None)
    keyword_3: Annotated[Optional[str], Field(default=None)]

    tags: list[str] = Field(default_factory=list)
    create_time: datetime = Field(default_factory=datetime.now)
    uuid: str = Field(default_factory=lambda: uuid4().hex)

    tag_list: list[str] = []  # This is OK
```

`default_factory` is useful when the default value is dynamically generated. For `list` and `dict`, it is okay to use literals `[]` and `{}`, because Pydantic will make a deep copy of it.


### Type conversion

For GET requests, input data are always of type `dict[str, str]`. For POST requests, though the client could send different types of values via JSON, like boolean and number, there are some types that are not representable in JSON, datetime for an example. When creating models, Pydantic will do proper type conversion. It is actually a part of validation, to ensure the client provides the correct data.

```python
class ConversionForm(BaseModel):
    int_value: int
    decimal_value: Decimal
    bool_value: bool
    datetime_value: datetime
    array_value: list[int]
    object_value: dict[str, int]

# Validate data returned by request.get_json()
ConversionForm.model_validate({
    'int_value': '10',
    'decimal_value': '10.24',
    'bool_value': 'true',
    'datetime_value': '2024-01-27 17:02:00',
    'array_value': [1, '2'],
    'object_value': {'key': '10'},
})
```

As a side note, if you are to create model with constructor, and pass a data type that does not match the model definition, mypy will raise an error:

```python
ConversionForm(int_value='10')
# error: Argument "int_value" to "ConversionForm" has incompatible type "str"; expected "int"
```

To fix this, you need to enable Pydantic's mypy plugin in `pyproject.toml`:

```toml
[tool.mypy]
plugins = [
    'pydantic.mypy',
]
```


## Data validation

Type conversion works as the first step of data validation. Pydantic makes sure the model it creates contains attributes with the correct type. For further validation, Pydantic provides some builtin validators, and users are free to create new ones.


### Builtin validators

Here are three ways to ensure `username` contains 3 to 10 characters, with builtin validators:

```python
# Field definition
from pydantic import BaseModel, Field

class UserForm(BaseModel):
    username: str = Field(min_length=3, max_length=10)

# Annotated type
from typing import Annotated

ValidName = Annotated[str, Field(min_length=3, max_length=10)]

class UserForm(BaseModel):
    username: ValidName

# Use annotated-types package, auto-installed by Pydantic
from annotated_types import MinLen, MaxLen, Len

ValidName = Annotated[str, MinLen(3), MaxLen(10)]
# Or
ValidName = Annotated[str, Len(3, 10)]
```

Some useful builtin validators are listed below. For annotated-types package, please check its [repository][4] for more.

* String constraints
    * `min_length`
    * `max_length`
    * `pattern`: Regular expression, e.g. `r'^[0-9]+$'`
* Numeric constraints
    * `gt`: Greater than
    * `lt`: Less than
    * `ge`: Greater than or equal to
    * `le`: Less than or equal to
* Decimal constraints
    * `max_digits`
    * `decimal_places`

In addition, Pydantic defines several special types for validation. For instance:

```python
from pydantic import PostiveInt

class Model(BaseModel):
    int_1: PositiveInt
    int_2: Annotated[int, Field(gt=0)]
```

`int_1` and `int_2` are equivalent, they both accept integer that is greater than 0. Other useful predefined types are:

* `NegativeInt`, `NonPositiveInt`, `NonNegativeFloat`, etc.
* `StrictInt`: Only accept integer value like `10`, `-20`. Raise error for string `"10"` or float `10.0`. [Strict mode][5] can be enabled on field level, model level, or per validation.
* `AwareDatetime`: Datetime must contain timezone information, e.g. `2024-01-28T07:58:00+08:00`
* `AnyUrl`: Accept a valid URL, and user can access properties like `scheme`, `host`, `path`, etc.
* `Emailstr`: Accept a valid email address. This requires an extra package, i.e. `pip intall "pydantic[email]"`
* `IPvAnyAddress`: Accept a valid IPv4 or IPv6 address.
* `Json`: Accept a JSON string and convert it to Python object. For example:

```python
from pydantic import Json

class SearchForm(BaseModel):
    tags: Json[list[str]]

form = SearchForm(tags='["a", "b", "c"]')
form.model_dump(mode='json')
# {"tags": ["a", "b", "c"]}
```

### Choices

Another common use case for validation is to only accept certain values for a field. This can be done with `Literal` type:

```python
from typing import Literal

class SearchForm(BaseModel):
    order_by: Literal['asc', 'desc'] = 'asc'

SearchForm(order_by='natural')
# ValidationError order_by Input should be 'asc' or 'desc'
```

Or `Enum`:

```python
from enum import Enum

class OrderBy(str, Enum):
    ASC = 'asc'
    DESC = 'desc'

class Category(IntEnum):
    BIG_DATA = 1
    PROGRAMMING = 2

class SearchForm(BaseModel):
    order_by: OrderBy
    category: Category

form = SearchForm(order_by='asc', category='2')
assert form.order_by == OrderBy.ASC
assert form.category == Category.PROGRAMMING
```

### Custom validator

As shown in the previous section, there are three ways to define a validator. But this time we want to apply custom logics *after* the default validation.

```python
# Field decorator
from pydantic import field_validator

class UserForm(BaseModel):
    username: str

    @field_validator('username')
    @classmethod
    def validate_username(cls, value: str) -> str:
        if len(value) < 3 or len(value) > 10:
            raise ValueError('should have 3 to 10 characters')
        return value

# Annotated type
from pydantic import AfterValidator

def validate_name(value: str) -> str:
    if len(value) < 3 or len(value) > 10:
        raise ValueError('should have 3 to 10 characters')
    return value

class UserForm(BaseModel):
    username: Annotated[str, AfterValidator(validate_name)]

# Model validator
from pydantic import model_validator

class UserForm(BaseModel):
    username: str

    @model_validator(mode='after')
    def validate_model(self) -> 'UserForm':
        if len(self.username) < 3 or len(self.username) > 10:
            raise ValueError('username should have 3 to 10 characters')
        return self
```


## Handle validation error

All validation errors, including the `ValueError` we raise in custom validator, are wrapped in Pydantic's `ValidationError`. So a common practice is to setup a global error handler for it. Take Flask for an instance:

```python
from flask import Response, jsonfiy
from pydantic import ValidationError

@app.errorhandler(ValidationError)
def handle_validation_error(error: ValidationError) -> tuple[Response, int]:
    detail = error.errors()[0]
    if detail['loc']:
        message = f'{detail["loc"][0]}: {detail["msg"]}'
    else:
        message = detail['msg']

    payload = {
        'code': 400,
        'message': message,
    }
    return jsonify(payload), 400
```

`ValidationError` provides full description of all errors. Here we only take the first error and return the field name and error message:

```
% http localhost:5000/create-user username=a password=password
HTTP/1.1 400 BAD REQUEST
Content-Type: application/json

{
    "code": 400,
    "message": "username: Value error, should have 3 to 10 characters"
}
```

To further customize the validation error, one can construct a `PydanticCustomError`:

```python
# In field validator
raise PydanticCustomError('bad_request', 'Invalid username', {'code': 40001})

# In error handler
if detail['type'] == 'bad_request':
    payload = {
        'code': detail['ctx']['code'],
        'message': detail['msg'],
    }
```


### Validate routing variables

Pydantic provides a decorator to validate function calls. This can be used to validate Flask's routing variables as well. For instance, Flask accepts non-negative integer, but Pydantic requires it to be greater than 0.

```python
from pydantic import validate_call, PositiveInt

@app.get('/user/<int:user_id>')
@validate_call
def get_user_by_id(user_id: PositiveInt) -> dict:
    return {'id': user_id}
```

Validation result:

```
% http localhost:5000/user/0
HTTP/1.1 400 BAD REQUEST
Content-Type: application/json

{
    "code": 400,
    "message": "user_id: Input should be greater than 0"
}
```


## Integrate with OpenAPI

The quickest way is to use a framework that builds with Pydantic and OpenAPI, a.k.a. [FastAPI][6]. But if you are using a different framework, or maintaining an existing project, there are several options.


### Export model to JSON schema

Pydantic provides the facility to export models as JSON schema. We can write a Flask command to save them into a file:

```python
from pydantic.json_schema import models_json_schema

@app.cli.command()
def export_schema():
    _, schema = models_json_schema([
        (UserForm, 'validation'),
        (CreateUserResponse, 'serialization'),
    ])
    with open('schemas.json', 'w') as f:
        f.write(json.dumps(schema, indent=2))
```

The generated `schemas.json` would be:

```json
{
  "$defs": {
    "UserForm": {
      "type": "object",
      "properties": {
        "username": {
          "type": "string",
          "maxLength": 10,
          "minLength": 3
        },
        ...
      },
    },
    ...
  }
}
```

Then we create an `openapi.yaml` file to use these schemas:

```yaml
openapi: 3.0.2
info:
  title: Pydantic Demo
  version: 0.1.0
paths:
  /create-user:
    post:
      requestBody:
        content:
          application/json:
            schema:
              $ref: 'schemas.json#/$defs/UserForm'
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: 'schemas.json#/$defs/CreateUserResponse'
```

Open it in some OpenAPI viewer, e.g. OpenAPI extension for VS Code:

![OpenAPI](/images/api-pydantic/openapi.png)


### Create OpenAPI specification in Python

Install [openapi-pydantic][7], and define OpenAPI like a Pydantic model:

```python
from openapi_pydantic import OpenAPI
from openapi_pydantic.util import PydanticSchema, construct_open_api_with_schema_class

api = OpenAPI.model_validate({
    'info': {'title': 'Pydantic Demo', 'version': '0.1.0'},
    'paths': {
        '/create-user': {
            'post': {
                'requestBody': {'content': {'application/json': {
                    'schema': PydanticSchema(schema_class=UserForm),
                }}},
                'responses': {'200': {
                    'description': 'OK',
                    'content': {'application/json': {
                        'schema': PydanticSchema(schema_class=CreateUserResponse),
                    }},
                }},
            },
        },
    },
})
api = construct_open_api_with_schema_class(api)
print(api.model_dump_json(by_alias=True, exclude_none=True, indent=2))
```

The generated file is similar to the previous one, except that it is written in JSON and schemas are embedded in the `components` section.


### Decorate API endpoints

[SpecTree][8] provides facilities to decorate Flask view methods with Pydantic models. It generates OpenAPI docs in http://localhost:5000/apidoc/swagger.

```python
app = Flask(__name__)
spec = SpecTree('flask', annotations=True)
spec.register(app)

@app.post('/create-user')
@spec.validate(resp=Response(HTTP_200=CreateUserResponse))
def create_user(json: UserForm) -> CreateUserResponse:
    user_orm = UserOrm(username=json.username, password=json.username)
    return CreateUserResponse(id=user_orm.id)
```

* Pydantic `BaseModel` needs to be imported from `pydantic.v1`, for compatibility reason.
* Validation error is returned to client with HTTP status 422 and detailed information:

```
% http localhost:5000/create-user username=a password=password
HTTP/1.1 422 UNPROCESSABLE ENTITY
Content-Type: application/json

[
    {
        "ctx": {
            "limit_value": 3
        },
        "loc": [
            "username"
        ],
        "msg": "ensure this value has at least 3 characters",
        "type": "value_error.any_str.min_length"
    }
]
```

## References
* https://docs.pydantic.dev/latest/concepts/models/
* https://fastapi.tiangolo.com/tutorial/
* https://swagger.io/docs/specification/data-models/


[1]: https://pydantic.dev/
[2]: https://jizhang.github.io/blog/2024/01/19/python-static-type-check/
[3]: https://sqlmodel.tiangolo.com/
[4]: https://github.com/annotated-types/annotated-types
[5]: https://docs.pydantic.dev/latest/concepts/strict_mode/
[6]: https://fastapi.tiangolo.com/
[7]: https://github.com/mike-oakley/openapi-pydantic
[8]: https://github.com/0b01001001/spectree
[9]: https://httpie.io/
