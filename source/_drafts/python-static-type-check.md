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

![Mypy](/images/python-typing/mypy.png)

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

There are basic types like `str`, `int`, and collection types like `list`, `dict`. We can even define the type of their elements.

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

The check works as expected: `items` is a `list`, so it cannot be assigned otherwise; `nums` is a list of numbers, no string is allowed; the value of `ages` is also restricted. Look carefully at the first error message, we can see `list` is equivalent to `list[Any]`, where `Any` is also defined in `typing` module, which means literally any type. For instance, if a function argument is not given a type hint, it is defined as `Any` and can accept any type of value.

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

From the code above we can see that it does take some effort to write Python with type hints, so why is it peferrable anyway? Actually the merits can be drawn from many other statically typed languages like Go and Java:

* Errors can be found at compile time, or even earlier if you are coding in an IDE.
* [Studies][2] show that TypeScript or Flow can reduce the number of bugs by 15%.
* Static typing can improve the readability and maintainability of program.
* Type hints may have a positive impact on performance.

Before we dive into details, let's differentiate between strong/weak typing and static/dynamic typing.

![Categories of typing](/images/python-typing/categories.png)

Static/dynamic typing is easier to tell apart. Static typing validates variable types at compile time, such as Go, Java and C, while dynamic typing checks at runtime, like Python, JavaScript and PHP. Strong/weak typing, on the other hand, depends on the extent of implicit conversion. For instance, JavaScript is the least weakly typed language because all types of values can be added to each other. It is the language interpreter than does the implict conversion, so that number can be added to array, string to object, etc. PHP is another example of weakly typed language, in that string can be added to number, but a warning will be reported. Python, on the contrary, is strongly typed because this operation will immediately raise a `TypeError`.

Back to the advantages of static typing. For Python, type hints can improve the readability of code. The following snippet defines the function arguments with explict types, so that the checker would instantly warn your about a wrong call. Besides, type hints are also used by editor to provide informative and accurate autocomplete for invoking methods on an object. Python standard library is fully augmented with type hints, so you can input `some_str.` and choose from a list of methods of `str` object.

```python
from typing import Any, Optional, NewType

UserId = NewType('UserId', int)

def send_request(request_data: Any,
                 headers: Optional[dict[str, str]],
                 user_id: Optional[UserId] = None,
                 as_json: bool = True):
    ...
```

For some languages, type hints also boost the performance. Take Clojure for an example:

```clojure
(defn len [x]
  (.length x))

(defn len2 [^String x]
  (.length x))

user=> (time (reduce + (map len (repeat 1000000 "asdf"))))
"Elapsed time: 3007.198 msecs"
4000000
user=> (time (reduce + (map len2 (repeat 1000000 "asdf"))))
"Elapsed time: 308.045 msecs"
4000000
```

The untyped version of `len` costs about ten times longer. Because Clojure is designed as a dynamically typed language too, and uses reflection to determine the type of variable. This process is rather slow, so type hint works well in performance critical scenarios. But this is not true for Python, because type hints are completely ignored at runtime.

Some other languages also start to adopt static typing. TypeScript, a superset of JavaScript with syntax for types:

```typescript
const isDone: boolean = false
const decimal: number = 6
const color: string = 'blue'

const listA: number[] = [1, 2, 3]
const listB: Array<number> = [1, 2, 3]

function add(x: number, y: number): number {
  return x + y
}
```

And the Hack programming language, which is PHP with static typing and a lot of new features:

```php
<?hh
class MyClass {
  const int MyConst = 0;
  private string $x = '';

  public function increment(int $x): int {
    $y = $x + 1;
    return $y;
  }
}
```

That being said, whether to adopt static typing for Python depends on the size of your project, or how formal it is. Luckily Python provides a gradual way of adopting static typing, so you do not need to add all type hints in one go. This approach will be dicussed in the next section.


## Python static typing in details

### PEP

Every new feature in Python comes with a PEP. The PEPs related to static typing can be found in [this link][3]. Some of the important ones are:

* PEP 3107 Function Annotation (Python 3.0)
* PEP 484 Type Hints (Python 3.5)
* PEP 526 Syntax for Variable Annotations (Python 3.6)
* PEP 563 Postponed Evaluation of Annotations (Python 3.7)
* PEP 589 TypedDict (Python 3.8)
* PEP 585 Type Hinting Generics In Standard Collections (Python 3.9)
* PEP 604 Allow writing union types as X | Y (Python 3.10)

Python 3.0 introduces the annotation syntax for function arguments and return value, but it was not solely designed for type checking. From Python 3.5, a complete syntax for static typing is defined, `typing` module is added, and `mypy` is made the reference implementation for type checking. In later versions, more features are added like protocols, literal types, new callable syntax, etc., making static typing more powerful and delightful to use.

### Gradual typing

One that that never changes is that static typing is an opt-in, meaning you can apply it to the whole project or only some of the modules. As a result, you can progressively add type hints to certain parts of the program, even just a single function. Because in the default setting, mypy will only check functions that has at least one type hint in its signature:

```python
# Check
def greeting(name, age: int): ...
def greeting(name, age) -> str: ...

# Not check
def greeting(name, age): ...
```

For untyped argument, like `name` in the first `greeting`, it is considered as `Any` type, which means you can pass any value as `name`, and use it for any operations. It is different from `object` type though. Say you define an argument as `item: object` and try to invoke `item.foo()`, mypy will complain that `object` has no attribute `foo`. So if you are not sure what the type of a variable is, give it `Any` or simply leave it blank.

```python
# Check
def greeting() -> None: ...

# Not check
def greeting(): ...
```

Another common mistake is for functions without arguments and return value. We have to add `None` as the return type, otherwise mypy will silently skip it.


### Type hints

There are two ways to compose type hints: annotation and stub file.

```python
# Annotation
class DummyList:
    def __init__(self) -> None:
        self.elements: list[int] = []

    def add(self, element: int) -> None:
        self.elements.append(element)

# Code without type hints, filename: dummy_list.py
class DummyList:
    def __init__(self):
        self.elements = []

    def add(self, element):
        self.elements.append(element)

# Add type hints with stub file, filename: dummy_list.pyi
class DummyList:
    elements: list[int]
    def __init__(self) -> None: ...
    def add(self, element: int): ...
```

`...` or [Ellipsis][8] is a valid Python expression, and a conventional way to leave out implementation. Stub files are used to add type hints to existing codebase without changing its source. For instance, Python standard library is fully typed with stub files, hosted in a repository called [typeshed][4]. There are other third-party libraries in this repo too, and they are all released as separate packages in PyPI, prefixed with `types-`, like `types-requests`. They need to be installed explicitly, otherwise mypy would complain that it cannot find type definitions for these libraries. Fortunately, a lot of popular libraries have embraced static typing and do not require external stub files.

Mypy provides a nice [cheat sheet][5] for basic usage of type hints, and Python [documentation][6] contains the full description. Here're the entries that I find most helpful:

```python
# Basic types
x: int = 1
x: float = 1.0
x: bool = True
x: str = 'test'
x: bytes = b'test'

# Collection types
x: tuple[int, str, float] = (3, 'yes', 7.5)
x: list[int] = [1, 2, 3]
x: set[int] = {6, 7}
x: dict[str, float] = {'field', 2.0}

# Sepcial types
from typing import Any, Union, Optional, Callable

x: Any = 1
x = 'test'

x: int | str = 1  # Equavalent to x: Union[int, str]
x = 'test'

x: Optional[str] = some_function()
x.lower()  # Error: Optional[str] has no attribute lower
if x is not None:
    x.lower()  # OK

def f(num1: int, my_float: float = 3.5) -> float:
    return num1 + my_float
x: Callable[[int, float], float] = f
```

One of my favorite types is `Optional`, since it solves the problem of `None`. Mypy will raise error if you fail to guard against a nullable variable. `if x is not None` is also a way of [type narrowing][7], meaning mypy will consider `x` as `str` in the `if` block. Another useful type narrowing technique is `isinstance`.

Python classes are also types. Mypy understands inheritance, and class types can be used in collections, too.

```python
class Animal:
    pass

class Cat(Animal):
    pass

animal: Animal = Cat()
cats: list[Cat] = []

# Type alias
CatList = list[Cat]
cats: CatList = []

# Forward reference
class Dog(Animal):
    @staticmethod
    def create() -> 'Dog':
        return Dog()

    @staticmethod
    def get_list() -> list['Dog']:
        return []
```

Type alias is useful for shortening the type definition. And notice the quotes around `Dog`. It is called forward reference, that allows you to refer to a type that has not yet been fully defined. In a future version, possible Python 3.12, the quotes may be omitted.

Another useful utility from `typing` module is `TypedDict`. `dict` is a frequently used data structure in Python, so it would be nice to explicitly define the fields and types in it.

```python
from typing import TypedDict

class Point(TypedDict):
    x: int
    y: int

p1: Point = {'x': 1, 'y': 2}
p2 = Point(x=1, y=2)
```

`TypedDict` is like a regular `dict` at runtime, only the type checker would see the difference. Another option is to use Python [dataclass][9] to define DTO (Data Transfer Object). Mypy has [full support][10] for it, too.


## Advanced features

### Generics

`list` is a generic class, and the `str` in `list[str]` is called a type parameter. So generics are useful when the class is a kind of container, and does not care about the type of elements it contains. We can easily write a generic class on our own. Say we want to wrap a variable of arbitary type and log a message when its value is changed.

```python
from typing import TypeVar, Generic

T = TypeVar('T')

class LoggedVar(Generic[T]):
    def __init__(self, value: T):
        self.value = value

    def set(self, new: T):
        print(f'Set value to {new}, previous value is {self.value}')
        self.value = new

v = LoggedVar[int](1)
v.set(2)
```

We can define functions that deal with generic classes. For instance, to return the first element of any sequence-like data structure:

```python
from typing import TypeVar
from collections.abc import Sequence

T = TypeVar('T')

def first(seq: Sequence[T]) -> T:
    return seq[0]

print(first([1, 2, 3]))
print(first('abc'))
```

`Sequence` is an abstract base class, which we will discuss in the next section. `list` and `str` are both subclasses of `Sequence`, so they can be accepted by the function `first`.

Type parameter can also have a bound, meaning it must be a sublcass of a particular type:

```python
from typing import TypeVar
from collections.abc import Sized

T = TypeVar('T', bound=Sized)

def longer(x: T, y: T) -> T:
    if len(x) > len(y):
        return x
    return y

print(longer([1], [1, 2]))
print(longer([1], {1, 2}))
print(longer([1], 'ab'))
```

`list`, `set`, and `str` are all subclasses of `Sized`, in that they all have a `__len__` method, so they can be passed to `longer` and `len` without problem.

### Abstract base class

`Sequence` and `Sized` are both abstract base classes, or ABC. As the name indicates, the class contains abstract methods and is supposed to be inherited from. There are plenty of predefined collection ABCs in [collections.abc][11] module, and they form a hierarchy of collection types in Python:

![Python collection types](/images/python-typing/collection-types.png)

Mypy understands ABC, so it is a good practice to declare function arguments with a more general type like `Iterable[int]`, so that both `list[int]` and `set[int]` are acceptable. To write a custom class that behaves like `Iterable`, subclass it and provide the required methods.

```python
from collections.abc import Sized, Iterable, Iterator

class Bucket(Sized, Iterable[int]):
    def __len__(self) -> int:
        return 0

    def __iter__(self) -> Iterator[int]:
        return iter([])

bucket: Iterable[int] = Bucket()
```

### Duck typing

In the previous code listing, what if I remove the base class:

```python
class Bucket:
    def __iter__(self) -> Iterator[int]:
        return iter([])

bucket: Iterable[int] = Bucket()  # Error?
```

Is the class instance still assignable to `Iterable[int]`? The answer is yes, because `Bucket` class would behave like an `Iterable[int]`, in that it contains a method `__iter__` and its return value is of correct type. It is called duck typing: If it walks like a duck and quacks like a duck, then it must be a duck. Duck typing only works for simple ABCs, like `Iterable`, `Collection`. In Python, there is a dedicated name for this feautre, [Protocol][12]. Simply put, if the class defines required methods, mypy would consider it as the corresponding type.

```python
# Sized
def __len__(self) -> int: ...

# Container[T]
def __contains__(self, x: object) -> bool: ...

# Collection[T]
def __len__(self) -> int: ...
def __iter__(self) -> Iterator[T]: ...
def __contains__(self, x: object) -> bool: ...
```

It is also possible to define your own Protocol:

```python
from typing import Protocol
from collections.abc import Iterable

class Closeable(Protocol):
    def close(self) -> None: ...

class Resource:
    def close(self):
        self.file.close()

def close_all(resources: Iterable[Closeable]):
    for r in resources:
        r.close()

close_all([Resource()])  # OK
```

### Runtime validation



## References
* https://docs.python.org/3.10/library/typing.html
* https://mypy.readthedocs.io/en/stable/index.html
* https://typing.readthedocs.io/en/latest/
* https://wphomes.soic.indiana.edu/jsiek/what-is-gradual-typing/
* https://blog.zulip.com/2016/10/13/static-types-in-python-oh-mypy/


[1]: https://mypy-lang.org/
[2]: https://softwareengineering.stackexchange.com/questions/59606/is-static-typing-worth-the-trade-offs/371369#371369
[3]: https://peps.python.org/topic/typing/
[4]: https://github.com/python/typeshed
[5]: https://mypy.readthedocs.io/en/stable/cheat_sheet_py3.html
[6]: https://docs.python.org/3.10/library/typing.html
[7]: https://mypy.readthedocs.io/en/stable/type_narrowing.html
[8]: https://docs.python.org/3/library/constants.html#Ellipsis
[9]: https://docs.python.org/3.10/library/dataclasses.html
[10]: https://mypy.readthedocs.io/en/stable/additional_features.html#dataclasses
[11]: https://docs.python.org/3.10/library/collections.abc.html
[12]: https://docs.python.org/3.10/library/typing.html#typing.Protocol
