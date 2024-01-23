---
title: Define API Data Models with Pydantic
categories: [Programming]
tags: [python, pydantic, flask, openapi]
---

<!-- more -->

* Define response model
    * Installation, mypy plugin
    * Python typing
    * From sqlalchemy
    * Custom serializer, e.g. datetime
    * Computed fields
    * Alias, snake_case to camelCase
    * Nested
    * Exclude
    * TypedDict, dataclass
    * Context?
* Define request model
    * Modeling query string
    * Validate route variables
    * Custom deserializer
    * Default value, default factory
    * Required fields
    * Alias
    * Type conversion, datetime
    * Constraints
        * String, number, decimal
        * Choices
        * Annotation, Field, Gt
    * Custom validator
    * Validation error
    * Versus wtform, marshmallow
* Integrate with SQLAlchemy
    * SQLModel
* OpenAPI
    * JSON Schema
    * Example data
    * FastAPI

## References
* https://docs.pydantic.dev/latest/concepts/models/
