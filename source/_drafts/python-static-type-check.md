---
title: Python Static Type Check
categories: [Programming]
tags: [python, mypy]
---

Python is by design a dynamically typed programming language. It is flexible and easy to write. But as the project size grows, there will be more interactions between functions, classes and modules, and we often make mistakes like passing wrong types of arguments or assuming different return types from function calls. Worse still, these mistakes can only be spotted in runtime, and are likely to cause production bugs. Is it possible for Python to support static typing like Java and Go, checking errors in compile time, while remaining to be easy to use? Fortunately, from Python 3.5 on, it supports an optional syntax, or type hints, for static type check, and many tools are built around this feature. This article covers the following topics:

* A quick start to do static type check in Python.
* Why do we need static typing.
* Python type hints in detail.
* Other advanced features.

![Mypy](/images/mypy.png)

## Quick start

Static typing can be achieved by adding type hints to function arguments and return value, while using a tool like [mypy][1] to do the check. For instance:

```python
def greeting(name: str) -> str:
    return 'hello ' + name
```

Here the function `greeting` accepts an argument which is typed as `str`, and its return value is also typed `str`. Run `pip install mypy`, and then check the file:

```
% mypy quickstart.py
Success: no issues found in 1 source file
```

Clearly this simple function would pass the check. Let's add some erroneous code:

```python
def greeting(name: str) -> str:
    real_name = name + 1
    return 'hello ' + real_name

greeting(1)
greeting('world') + 1
```

There will be plenty of errors found by mypy:

```
% mypy quickstart.py
quickstart.py:2: error: Unsupported operand types for + ("str" and "int")  [operator]
quickstart.py:5: error: Argument 1 to "greeting" has incompatible type "int"; expected "str"  [arg-type]
quickstart.py:6: error: Unsupported operand types for + ("str" and "int")  [operator]
Found 3 errors in 1 file (checked 1 source file)
```

The error messages are pretty clear. Usually we use pre-commit hook and CI to ensure everything checked in Git or merged in `master` passes `mypy`.

<!-- more -->

Type hints can also be applied to local variables. But most of the time, `mypy` is able to *infer* the type from the value.

```python
def greeting(name: str) -> str:
    real_name = 'hello ' + name
    number: int = real_name
    return number
```

`real_name` would be inferred as `str` type, so when it is assigned to `number`, an `int` typed variable, error occurs. The return value also includes an error.

```
% mypy quickstart.py
quickstart.py:3: error: Incompatible types in assignment (expression has type "str", variable has type "int")  [assignment]
quickstart.py:4: error: Incompatible return value type (got "int", expected "str")  [return-value]
Found 2 errors in 1 file (checked 1 source file)
```

There are basic types like `str`, `int`, `float`, `bool`, `bytes`. There are also collection types like `tuple`, `list`, `dict`, `set`, and we can even define the type of their elements.

```python
items: list = 0

nums: list[int] = []
nums.append('text')

ages: dict[str, int] = {}
ages['John'] = '30'
```

You may see some code written as `List[int]` or `Dict[str, int]`, where `List` is imported from the `typing` module. This is because before Python 3.9, `list` and other builtins do not support subscripting `[]`. This article's examples are based on Python 3.10.

```
% mypy quickstart.py
quickstart.py:1: error: Incompatible types in assignment (expression has type "int", variable has type "list[Any]")  [assignment]
quickstart.py:4: error: Argument 1 to "append" of "list" has incompatible type "str"; expected "int"  [arg-type]
quickstart.py:7: error: Incompatible types in assignment (expression has type "str", target has type "int")  [assignment]
Found 3 errors in 1 file (checked 1 source file)
```

The check works as expected: `items` is a `list`, so it cannot be assigned otherwise; `nums` is a list of numbers, no string is allowed; the value of `ages` is also restricted. Look carefully at the first error message, we can see `list` is equivalent to `list[Any]`, where `Any` is also defined in `typing` module, which means literally any type. For instance, if a function argument is not given a type hint, it is defined as `Any` and can accepts any type of value.

Please remember, these checks do not happen at runtime. Python remains to be a dynamically typed language. If you need runtime validation, extra tools are required. We will discuss it in a later section.

The last example is defining types for class members:

```python
class Job:
    suffix: str

    def __init__(self, date: str, suffix: str):
        self.date = date

    def run(self) -> None:
        self.date + 1
        self.suffix + 1
```

Type hints could be applied either in class body or in constructor. Member functions are typed as normal.

```
% mypy quickstart.py
quickstart.py:8: error: Unsupported operand types for + ("str" and "int")  [operator]
quickstart.py:9: error: Unsupported operand types for + ("str" and "int")  [operator]
Found 2 errors in 1 file (checked 1 source file)
```


## Why do we need static typing?


## References
* https://docs.python.org/3.10/library/typing.html
* https://peps.python.org/topic/typing/
* https://realpython.com/python-type-checking/
* https://typing.readthedocs.io/en/latest/
* https://mypy.readthedocs.io/en/stable/cheat_sheet_py3.html
* https://mypy.readthedocs.io/en/stable/builtin_types.html


[1]: https://mypy-lang.org/
