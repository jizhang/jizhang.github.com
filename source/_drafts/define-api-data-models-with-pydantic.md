---
title: Define API Data Models with Pydantic
categories: [Programming]
tags: [python, pydantic, flask, openapi]
---

<!-- more -->

* Define response model
    * Installation, mypy plugin
    * Python typing
    * From sqlalchemy, sqlmodel
    * Custom serializer, e.g. datetime
    * Computed fields
    * Alias, snake_case to camelCase
    * Nested
    * Exclude
    * TypedDict, dataclass
    * Context: excluded field
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
