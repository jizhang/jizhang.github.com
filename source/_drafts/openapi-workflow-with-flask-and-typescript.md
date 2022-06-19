---
title: OpenAPI Workflow with Flask and TypeScript
categories: Programming
tags: [openapi, python, flask, typescript, vue]
---

[OpenAPI][1] has become the de facto standard of designing web APIs, and there are numerous tools developed around its ecosystem. In this article, I will demonstrate the workflow of using OpenAPI in both backend and frontend projects.

![OpenAPI 3.0](/images/openapi-workflow/openapi-3.0.png)

## API Server

There are [code first and design first][2] approaches when using OpenAPI, and here we go with code first approach, i.e. writing the API server first, add specification to the method docs, then generate the final OpenAPI specification. The API server will be developed with Python [Flask][3] framework and [apispec][4] library with [marshmallow][5] extension. Let's first install the dependencies:

```
Flask==2.1.2
Flask-Cors==3.0.10
Flask-SQLAlchemy==2.5.1
SQLAlchemy==1.4.36
python-dotenv==0.20.0
apispec[marshmallow]==5.2.2
apispec-webframeworks==0.5.2
```

<!-- more -->

### Get post list

We will develop a simple blog post list page like this:

![Blog post list](/images/openapi-workflow/blog-post-list.png)

The first API we implement is `get_post_list`:

```python
@app.get('/api/post/list')
def get_post_list() -> Response:
    page = int(request.args.get('page', '1'))
    sort = request.args.get('sort', 'desc')

    query = db.session.query(Post)
    if sort == 'desc':
        query = query.order_by(Post.updated_at.desc())

    query = query.offset((page - 1) * PAGE_SIZE).limit(PAGE_SIZE)
    posts = query.all()
    return jsonify(posts=post_schema.dump(posts, many=True))
```

It is a regular web API, that parses the GET parameters, construct a SQLAlchemy query with pagination and return the post list. The only thing special here is `post_schema`, which uses the marshmallow library to serialize post items. The schema is defined as:

```python
class PostSchema(Schema):
    id = fields.Integer()
    title = fields.String(required=True, validate=validate.Length(min=1))
    content = fields.String(required=True, validate=validate.Length(min=1))
    updated_at = fields.DateTime(format='%Y-%m-%d %H:%M:%S', dump_only=True)


post_schema = PostSchema()
```

The schema is not that different from the SQLAlchemy model, but with extra information about validation, format, and whether some field should be dumped to the output. We'll see how to use the schema to deserialize and validate the form data later in `save_post` API.

```python
class Post(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String)
    content = db.Column(db.String)
    updated_at = db.Column(db.DateTime)
```

## API Client

[1]: https://www.openapis.org/
[2]: https://swagger.io/blog/api-design/design-first-or-code-first-api-development/
[3]: https://flask.palletsprojects.com/
[4]: https://apispec.readthedocs.io/
[5]: https://marshmallow.readthedocs.io/
