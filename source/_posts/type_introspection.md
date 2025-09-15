---
title: When your Python code needs to understand itself
summary: "A practical journey through Python's inspect and typing modules - the detective tools that let your applications understand function signatures and types at runtime"
date: "2025-08-09"
tags:
    - Python
---

# When your code needs to read other code

Python's type system has evolved significantly over the years, and with it, powerful tools for introspection and type manipulation have emerged. If your application needs to understand function signatures and types at runtime. Whether you are building frameworks, validation systems, or dynamic APIs—Python's, `inspect` and `typing` modules become invaluable allies.

In this article, I will briefly explore the functions from these modules that I have been using to build type-aware applications, sharing some practical insights along the way.

## `inspect.signature()`, the function detective

The `inspect.signature()` function is literally a detective for your functions. It reveals everything about a function's parameters and their characteristics:

```python
import inspect

def example_function(user_id: int, name: str = "guest", is_active: bool = True):
    pass

sig = inspect.signature(example_function)

sig
# Return: <Signature (user_id: int, name: str = 'guest', is_active: bool = True)>
```

What makes this function powerful is that it returns a `Signature` object containing detailed information about each parameter:

```python
for param_name, param in sig.parameters.items():
    print(f"Parameter: {param_name}")
    print(f"Type: {param.annotation}")
    print(f"Default: {param.default}")
    print(f"Kind: {param.kind}")
    print("---")
```

This code outputs:

```text
Parameter: user_id
Type: <class 'int'>
Default: <class 'inspect._empty'>
Kind: POSITIONAL_OR_KEYWORD
---
Parameter: name
Type: <class 'str'>
Default: guest
Kind: POSITIONAL_OR_KEYWORD
---
Parameter: is_active
Type: <class 'bool'>
Default: True
Kind: POSITIONAL_OR_KEYWORD
---
```

The Signature object gives you access to:

- **Parameter names**: the names of all parameters
- **Type annotations**: the type hints for each parameter
- **Default values**: whether parameters have defaults and what they are
- **Parameter kinds**: `POSITIONAL_ONLY`, `POSITIONAL_OR_KEYWORD`, `VAR_POSITIONAL`, etc.

## `typing.get_type_hints()`, the type resolver

While `inspect.signature()` gives you parameter information, `typing.get_type_hints()` goes a step further by resolving all type hints, including forward references and string annotations:

```python
from typing import get_type_hints, Any

def process_data(data: list[str], config: dict[str, Any]) -> bool:
    pass

hints = get_type_hints(process_data)
print(hints)
# {'data': list[str], 'config': dict[str, Any], 'return': bool}
```

This function is particularly useful because:

- It returns a dictionary which makes it very easy to manipulate or parse
- It resolves forward references (types defined as strings)
- It handles complex generic types properly
- It includes the return type annotation
- It works with classes, not just functions

## `typing.get_origin()` and `typing.get_args()`

The typing module provides additional functions for working with generic types, which are essential for modern Python type annotations. These functions help you understand the structure of "nested" types like `list[str]`, `dict[str, int]`, or `Optional[int]`:

```python
from typing import get_origin, get_args, Optional, Union

# Working with list[str]
list_type = list[str]
print(get_origin(list_type))  # <class 'list'>
print(get_args(list_type))    # (<class 'str'>,)

# Working with dict[str, int]
dict_type = dict[str, int]
print(get_origin(dict_type))  # <class 'dict'>
print(get_args(dict_type))    # (<class 'str'>, <class 'int'>)

# Working with Optional[str] (which is Union[str, None])
optional_type = Optional[str]
print(get_origin(optional_type))  # typing.Union
print(get_args(optional_type))    # (<class 'str'>, <class 'NoneType'>)
```

These functions are quite useful and have helped me with **schema generation** by converting Python types to JSON Schema.

## Some tips from personal usage

By using these functions I've learned some important lessons:

**1. Use `get_type_hints()` instead of direct annotation access**: it properly resolves forward references and string annotations that would break raw `__annotations__` access.

**2. Handle `inspect.Parameter.empty` correctly**: this special value indicates that a parameter has no default value:

```python
if param.default is not inspect.Parameter.empty:
    print(f"Has default: {param.default}")
else:
    print("Required parameter")
```

**3. Consider performance**: introspection has a cost, so cache results when possible.

**4. Handle edge cases**: not all types can be easily introspected, so provide fallbacks for functions without perfect type annotations.

## Conclusion

The `inspect` and `typing` modules provide powerful tools for runtime type introspection and manipulation. By combining `inspect.signature()`, `typing.get_type_hints()`, `typing.get_origin()`, and `typing.get_args()`, I am currently trying to build end-to-end type checking and validation systems for python microservices. Let's see how far I can get and if I can make this, a reality.
