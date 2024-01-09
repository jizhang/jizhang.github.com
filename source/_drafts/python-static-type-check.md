---
title: Python Static Type Check
categories: [Programming]
tags: [python, mypy]
---

Python is by design a dynamically typed programming language. It is flexible and easy to write. But as the project size grows, there will be more interactions between functions, classes and modules, and we often make mistakes like passing wrong types of arguments or assuming different return types from function calls. Worse still, these mistakes can only be spotted in runtime, and are likely to cause production bugs. Is it possible for Python to support static typing like Java and Go, checking errors in compile time, while remaining to be easy to use? Fortunately, from Python 3.5 on, it supports an optional syntax, or type hints, for static type check, and many tools are built around this feature. This article covers the following topics:

* A quick start to do static type check in Python.
* Why do we need static typing?
* Python type hints in detail.
* Other advanced features.

![Mypy](/images/mypy.png)

## Quick start

<!-- more -->

## References
* https://docs.python.org/3.10/library/typing.html
* https://peps.python.org/topic/typing/
* https://realpython.com/python-type-checking/
* https://typing.readthedocs.io/en/latest/
* https://mypy.readthedocs.io/en/stable/cheat_sheet_py3.html
* https://mypy.readthedocs.io/en/stable/builtin_types.html
