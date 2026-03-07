# Python Interview Preparation — Complete Guide

> A comprehensive, self-contained reference covering all Python fundamentals you need for technical interviews.

---

## Table of Contents

1. [Data Types & Variables](#1-data-types--variables)
2. [Operators](#2-operators)
3. [Strings](#3-strings)
4. [Lists, Tuples, Sets & Dictionaries](#4-lists-tuples-sets--dictionaries)
5. [Control Flow](#5-control-flow)
6. [Functions](#6-functions)
7. [List Comprehensions & Generator Expressions](#7-list-comprehensions--generator-expressions)
8. [Object-Oriented Programming](#8-object-oriented-programming)
9. [Exception Handling](#9-exception-handling)
10. [File Handling](#10-file-handling)
11. [Iterators & Generators](#11-iterators--generators)
12. [Decorators](#12-decorators)
13. [Lambda, Map, Filter, Reduce](#13-lambda-map-filter-reduce)
14. [Modules & Packages](#14-modules--packages)
15. [args & kwargs](#15-args--kwargs)
16. [Scope & Closures (LEGB)](#16-scope--closures-legb)
17. [Shallow Copy vs Deep Copy](#17-shallow-copy-vs-deep-copy)
18. [Python Memory Management & Garbage Collection](#18-python-memory-management--garbage-collection)
19. [Multithreading vs Multiprocessing & the GIL](#19-multithreading-vs-multiprocessing--the-gil)
20. [Common Built-in Functions](#20-common-built-in-functions)
21. [Important Standard Library Modules](#21-important-standard-library-modules)
22. [Common Interview Questions & Answers](#22-common-interview-questions--answers)
23. [Python 2 vs Python 3 Key Differences](#23-python-2-vs-python-3-key-differences)
24. [Tips & Best Practices](#24-tips--best-practices)

---

## 1. Data Types & Variables

### Built-in Data Types

| Category    | Types                                      |
| ----------- | ------------------------------------------ |
| Numeric     | `int`, `float`, `complex`                  |
| Boolean     | `bool` (`True` / `False`)                  |
| Sequence    | `str`, `list`, `tuple`, `range`            |
| Set         | `set`, `frozenset`                         |
| Mapping     | `dict`                                     |
| Binary      | `bytes`, `bytearray`, `memoryview`         |
| None        | `NoneType`                                 |

### Mutable vs Immutable

| Mutable                   | Immutable                            |
| ------------------------- | ------------------------------------ |
| `list`, `dict`, `set`, `bytearray` | `int`, `float`, `str`, `tuple`, `frozenset`, `bytes` |

- **Mutable** objects can be changed in place (same `id()`).
- **Immutable** objects create a new object on modification.

```python
# Immutable example
a = "hello"
print(id(a))
a = a + " world"   # new object created
print(id(a))       # different id

# Mutable example
lst = [1, 2, 3]
print(id(lst))
lst.append(4)      # modified in place
print(id(lst))     # same id
```

### Dynamic Typing

Python is **dynamically typed** — you don't declare variable types; the interpreter infers them at runtime.

```python
x = 10        # int
x = "hello"   # now str — no error
```

### Type Checking & Casting

```python
type(42)          # <class 'int'>
isinstance(42, int)  # True
isinstance(True, int)  # True — bool is subclass of int

int("10")         # 10
float("3.14")     # 3.14
str(100)          # '100'
list("abc")       # ['a', 'b', 'c']
tuple([1,2,3])    # (1, 2, 3)
set([1,2,2,3])    # {1, 2, 3}
```

### `is` vs `==`

```python
a = [1, 2, 3]
b = [1, 2, 3]
a == b    # True  — same value
a is b    # False — different objects in memory

c = a
a is c    # True  — same object
```

> **Interview tip:** `is` checks identity (`id()`), `==` checks equality (`__eq__`).

### Integer Caching

CPython caches small integers **-5 to 256**.

```python
a = 256
b = 256
a is b   # True  (cached)

a = 257
b = 257
a is b   # False (not cached — different objects)
```

---

## 2. Operators

### Arithmetic

| Operator | Description         | Example         |
| -------- | ------------------- | --------------- |
| `+`      | Addition            | `5 + 3 → 8`    |
| `-`      | Subtraction         | `5 - 3 → 2`    |
| `*`      | Multiplication      | `5 * 3 → 15`   |
| `/`      | True Division       | `7 / 2 → 3.5`  |
| `//`     | Floor Division      | `7 // 2 → 3`   |
| `%`      | Modulus             | `7 % 2 → 1`    |
| `**`     | Exponentiation      | `2 ** 3 → 8`   |

### Comparison

`==`, `!=`, `<`, `>`, `<=`, `>=`

### Logical

`and`, `or`, `not`

```python
# Short-circuit evaluation
True or expensive_func()   # expensive_func() never called
False and expensive_func() # expensive_func() never called
```

### Bitwise

| Operator | Description | Example                |
| -------- | ----------- | ---------------------- |
| `&`      | AND         | `5 & 3 → 1`           |
| `\|`     | OR          | `5 \| 3 → 7`          |
| `^`      | XOR         | `5 ^ 3 → 6`           |
| `~`      | NOT         | `~5 → -6`             |
| `<<`     | Left shift  | `1 << 3 → 8`          |
| `>>`     | Right shift | `8 >> 2 → 2`          |

### Assignment Shortcuts

`+=`, `-=`, `*=`, `/=`, `//=`, `%=`, `**=`, `&=`, `|=`, `^=`, `<<=`, `>>=`

### Walrus Operator `:=` (Python 3.8+)

Assigns a value to a variable as part of an expression.

```python
# Without walrus
n = len(data)
if n > 10:
    print(n)

# With walrus
if (n := len(data)) > 10:
    print(n)
```

### Ternary Operator

```python
result = "even" if x % 2 == 0 else "odd"
```

### Operator Precedence (high → low)

`**` → `~, +, -` (unary) → `*, /, //, %` → `+, -` → `<<, >>` → `&` → `^` → `|` → `==, !=, <, >, <=, >=, is, in` → `not` → `and` → `or`

---

## 3. Strings

### Creation & Basics

```python
s1 = 'single quotes'
s2 = "double quotes"
s3 = '''triple
       quotes for multiline'''
s4 = r"raw string: \n not escaped"
s5 = f"f-string: {2 + 2}"   # 'f-string: 4'
```

Strings are **immutable sequences** of Unicode characters.

### Common Methods

```python
s = "Hello, World!"

s.upper()           # 'HELLO, WORLD!'
s.lower()           # 'hello, world!'
s.title()           # 'Hello, World!'
s.capitalize()      # 'Hello, world!'
s.swapcase()        # 'hELLO, wORLD!'

s.strip()           # Remove leading/trailing whitespace
s.lstrip()          # Remove leading whitespace
s.rstrip()          # Remove trailing whitespace

s.split(", ")       # ['Hello', 'World!']
", ".join(["a","b"])# 'a, b'

s.find("World")     # 7  (returns -1 if not found)
s.index("World")    # 7  (raises ValueError if not found)
s.count("l")        # 3

s.replace("World", "Python")  # 'Hello, Python!'

s.startswith("Hello")  # True
s.endswith("!")        # True

s.isdigit()         # False
s.isalpha()         # False (has comma, space, !)
s.isalnum()         # False
s.isspace()         # False

s.zfill(20)         # '0000000Hello, World!'
s.center(20, '-')   # '---Hello, World!----'
s.ljust(20, '-')    # 'Hello, World!-------'
s.rjust(20, '-')    # '-------Hello, World!'
```

### Slicing

```python
s = "Python"
s[0]      # 'P'
s[-1]     # 'n'
s[1:4]    # 'yth'
s[:3]     # 'Pyt'
s[3:]     # 'hon'
s[::2]    # 'Pto'
s[::-1]   # 'nohtyP' — reverse
```

### String Formatting

```python
name, age = "Alice", 30

# f-string (preferred, 3.6+)
f"Name: {name}, Age: {age}"

# .format()
"Name: {}, Age: {}".format(name, age)
"Name: {n}, Age: {a}".format(n=name, a=age)

# % formatting (old style)
"Name: %s, Age: %d" % (name, age)

# Format specifiers
f"{3.14159:.2f}"      # '3.14'
f"{1000000:,}"        # '1,000,000'
f"{42:08d}"           # '00000042'
f"{255:#x}"           # '0xff'
f"{'left':<10}"       # 'left      '
f"{'right':>10}"      # '     right'
f"{'center':^10}"     # '  center  '
```

### String Interning

Python interns (reuses) certain strings (identifiers, small strings) for performance.

```python
a = "hello"
b = "hello"
a is b   # True — interned

a = "hello world!"
b = "hello world!"
a is b   # May be False — not always interned
```

---

## 4. Lists, Tuples, Sets & Dictionaries

### Lists

Ordered, mutable, allows duplicates.

```python
lst = [1, 2, 3, "four", [5, 6]]

# Access
lst[0]        # 1
lst[-1]       # [5, 6]
lst[1:3]      # [2, 3]

# Modification
lst.append(7)           # [1, 2, 3, 'four', [5, 6], 7]
lst.insert(0, 0)        # [0, 1, 2, 3, 'four', [5, 6], 7]
lst.extend([8, 9])      # appends multiple items
lst += [10]             # same as extend for single list

lst.remove("four")      # removes first occurrence
lst.pop()               # removes & returns last item
lst.pop(0)              # removes & returns item at index 0
del lst[1]              # delete by index

lst.sort()              # in-place sort (all elements must be comparable)
lst.sort(reverse=True)  # descending
sorted(lst)             # returns new sorted list

lst.reverse()           # in-place reverse
lst[::-1]               # returns new reversed list

lst.index(3)            # index of first occurrence of 3
lst.count(2)            # count occurrences of 2
len(lst)                # length

lst.copy()              # shallow copy
lst.clear()             # remove all items
```

### Tuples

Ordered, immutable, allows duplicates. Faster than lists; can be used as dict keys.

```python
t = (1, 2, 3)
t2 = 1, 2, 3           # packing (parentheses optional)
t3 = (42,)              # single-element tuple — comma required!

a, b, c = t             # unpacking
a, *rest = (1,2,3,4)    # a=1, rest=[2,3,4]

t.count(2)              # 1
t.index(3)              # 2

# Named tuples
from collections import namedtuple
Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)
p.x   # 3
p.y   # 4
```

### Sets

Unordered, mutable, **no duplicates**. Elements must be **hashable** (immutable).

```python
s = {1, 2, 3, 3}       # {1, 2, 3}
s2 = set([1, 2, 3])

s.add(4)
s.discard(2)            # no error if missing
s.remove(3)             # KeyError if missing
s.pop()                 # remove & return arbitrary element

# Set operations
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

a | b       # union:        {1, 2, 3, 4, 5, 6}
a & b       # intersection:  {3, 4}
a - b       # difference:    {1, 2}
a ^ b       # symmetric diff:{1, 2, 5, 6}

a.issubset(b)       # False
a.issuperset(b)     # False
a.isdisjoint(b)     # False (they share elements)

# frozenset — immutable set, can be used as dict key
fs = frozenset([1, 2, 3])
```

### Dictionaries

Ordered (Python 3.7+), mutable, key-value pairs. Keys must be **hashable**.

```python
d = {"name": "Alice", "age": 30}
d2 = dict(name="Alice", age=30)

# Access
d["name"]             # 'Alice'
d.get("name")         # 'Alice'
d.get("salary", 0)    # 0 (default if key missing)

# Modify
d["age"] = 31
d["city"] = "NYC"     # add new key
d.update({"age": 32, "job": "dev"})

# Remove
del d["city"]
d.pop("job")          # returns value & removes key
d.pop("xyz", None)    # returns None if missing
d.popitem()           # removes & returns last inserted (k,v)

# Iteration
d.keys()              # dict_keys([...])
d.values()            # dict_values([...])
d.items()             # dict_items([(k,v), ...])

for k, v in d.items():
    print(k, v)

# Dictionary comprehension
squares = {x: x**2 for x in range(6)}

# Merging (Python 3.9+)
merged = d | {"new_key": "val"}

# defaultdict
from collections import defaultdict
dd = defaultdict(list)
dd["fruits"].append("apple")  # no KeyError

# Counter
from collections import Counter
Counter("abracadabra")  # Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})

# OrderedDict (preserves insertion order explicitly, mostly for Python < 3.7)
from collections import OrderedDict
```

### Comparison Table

| Feature       | List    | Tuple    | Set       | Dict         |
| ------------- | ------- | -------- | --------- | ------------ |
| Ordered       | ✅      | ✅       | ❌        | ✅ (3.7+)    |
| Mutable       | ✅      | ❌       | ✅        | ✅           |
| Duplicates    | ✅      | ✅       | ❌        | Keys: ❌     |
| Indexed       | ✅      | ✅       | ❌        | By key       |
| Hashable      | ❌      | ✅       | ❌        | Keys must be |

---

## 5. Control Flow

### `if` / `elif` / `else`

```python
if x > 0:
    print("positive")
elif x == 0:
    print("zero")
else:
    print("negative")
```

### `match` / `case` (Python 3.10+ — Structural Pattern Matching)

```python
match status:
    case 200:
        print("OK")
    case 404:
        print("Not Found")
    case _:
        print("Unknown")
```

### `for` Loop

```python
for i in range(5):          # 0, 1, 2, 3, 4
    print(i)

for i in range(2, 10, 3):  # 2, 5, 8
    print(i)

for char in "hello":
    print(char)

for i, val in enumerate(["a", "b", "c"]):
    print(i, val)           # 0 a, 1 b, 2 c

for a, b in zip([1,2], [3,4]):
    print(a, b)             # 1 3, 2 4
```

### `while` Loop

```python
while condition:
    # ...
    if should_stop:
        break
    if should_skip:
        continue
```

### `for/while … else`

The `else` block runs only if the loop **completes without `break`**.

```python
for n in range(2, 10):
    for x in range(2, n):
        if n % x == 0:
            break
    else:
        print(f"{n} is prime")
```

### `pass`, `break`, `continue`

- **`pass`** — no-op placeholder
- **`break`** — exit innermost loop
- **`continue`** — skip to next iteration

### Falsy Values

The following evaluate to `False` in boolean context:

`None`, `False`, `0`, `0.0`, `0j`, `""`, `[]`, `()`, `{}`, `set()`, `range(0)`, objects with `__bool__()` returning `False` or `__len__()` returning `0`.

Everything else is **truthy**.

---

## 6. Functions

### Defining & Calling

```python
def greet(name, greeting="Hello"):
    """Docstring: Returns a greeting string."""
    return f"{greeting}, {name}!"

greet("Alice")                # 'Hello, Alice!'
greet("Bob", greeting="Hi")  # 'Hi, Bob!'
```

### Return Values

```python
def divide(a, b):
    return a // b, a % b    # returns a tuple

quotient, remainder = divide(10, 3)
```

### Arguments

```python
# Positional
def f(a, b): ...

# Keyword
f(b=2, a=1)

# Default values (evaluated ONCE at definition time!)
def f(lst=[]):       # DANGEROUS — shared mutable default
    lst.append(1)
    return lst

f()  # [1]
f()  # [1, 1] — same list object!

# Safe pattern
def f(lst=None):
    if lst is None:
        lst = []
    lst.append(1)
    return lst

# Positional-only (/) and keyword-only (*) — Python 3.8+
def f(pos_only, /, normal, *, kw_only):
    ...

f(1, 2, kw_only=3)      # OK
f(pos_only=1, ...)       # Error
f(1, 2, 3)               # Error — kw_only must be keyword
```

### First-Class Functions

Functions are objects — they can be assigned to variables, passed as arguments, and returned.

```python
def square(x):
    return x ** 2

func = square
func(5)   # 25

def apply(f, value):
    return f(value)

apply(square, 4)  # 16
```

### Docstrings

```python
def my_func():
    """This is a one-line docstring."""
    pass

def complex_func(a, b):
    """
    Summary line.

    Args:
        a (int): First number.
        b (int): Second number.

    Returns:
        int: Sum of a and b.
    """
    return a + b

# Access docstring
complex_func.__doc__
help(complex_func)
```

### Type Hints (Python 3.5+)

```python
def add(a: int, b: int) -> int:
    return a + b

# Type hints are NOT enforced at runtime — they are for documentation and tooling (mypy).
from typing import List, Dict, Optional, Tuple, Union

def process(items: List[str]) -> Dict[str, int]:
    ...

def find(name: str) -> Optional[str]:   # can return str or None
    ...
```

---

## 7. List Comprehensions & Generator Expressions

### List Comprehension

```python
squares = [x**2 for x in range(10)]
evens = [x for x in range(20) if x % 2 == 0]
pairs = [(x, y) for x in range(3) for y in range(3)]

# Nested
matrix = [[1,2,3],[4,5,6],[7,8,9]]
flat = [num for row in matrix for num in row]  # [1,2,3,4,5,6,7,8,9]
```

### Dict Comprehension

```python
{k: v for k, v in [("a", 1), ("b", 2)]}
{x: x**2 for x in range(5)}
```

### Set Comprehension

```python
{x % 3 for x in range(10)}   # {0, 1, 2}
```

### Generator Expression

Like list comprehension but **lazy** — yields one item at a time, saving memory.

```python
gen = (x**2 for x in range(1000000))
next(gen)   # 0
next(gen)   # 1

sum(x**2 for x in range(1000000))  # no intermediate list created
```

---

## 8. Object-Oriented Programming

### Classes & Objects

```python
class Dog:
    # Class variable (shared by all instances)
    species = "Canis familiaris"

    def __init__(self, name, age):
        # Instance variables
        self.name = name
        self.age = age

    def bark(self):
        return f"{self.name} says Woof!"

    def __str__(self):
        return f"Dog({self.name}, {self.age})"

    def __repr__(self):
        return f"Dog('{self.name}', {self.age})"

dog = Dog("Rex", 5)
dog.bark()        # 'Rex says Woof!'
print(dog)        # Dog(Rex, 5) — uses __str__
Dog.species       # 'Canis familiaris'
```

### `self`

- **`self`** is a reference to the current instance. It's passed automatically but must be the first parameter in instance methods.
- It's a convention, not a keyword — you could name it anything (but don't).

### Inheritance

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        raise NotImplementedError

class Cat(Animal):
    def speak(self):
        return f"{self.name} says Meow!"

class Dog(Animal):
    def speak(self):
        return f"{self.name} says Woof!"

cat = Cat("Whiskers")
cat.speak()  # 'Whiskers says Meow!'
```

### Multiple Inheritance & MRO

```python
class A:
    def greet(self):
        return "A"

class B(A):
    def greet(self):
        return "B"

class C(A):
    def greet(self):
        return "C"

class D(B, C):
    pass

D().greet()      # 'B'
D.__mro__        # (D, B, C, A, object) — Method Resolution Order (C3 linearization)
```

### `super()`

Calls the parent class method following MRO.

```python
class Parent:
    def __init__(self, name):
        self.name = name

class Child(Parent):
    def __init__(self, name, age):
        super().__init__(name)
        self.age = age
```

### Encapsulation (Access Modifiers)

Python doesn't have true access modifiers; it uses **naming conventions**:

| Convention    | Meaning            | Example         |
| ------------- | ------------------ | --------------- |
| `name`        | Public             | `self.name`     |
| `_name`       | Protected (hint)   | `self._name`    |
| `__name`      | Private (mangled)  | `self.__name`   |

```python
class MyClass:
    def __init__(self):
        self.public = 1
        self._protected = 2
        self.__private = 3

obj = MyClass()
obj.public          # 1
obj._protected      # 2 (accessible, but convention says don't)
obj.__private       # AttributeError
obj._MyClass__private  # 3 (name mangling)
```

### Properties

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius cannot be negative")
        self._radius = value

    @property
    def area(self):
        import math
        return math.pi * self._radius ** 2

c = Circle(5)
c.radius         # 5
c.radius = 10    # uses setter
c.area           # 314.159...
```

### Class Methods & Static Methods

```python
class MyClass:
    count = 0

    def __init__(self):
        MyClass.count += 1

    @classmethod
    def get_count(cls):
        """Receives the class as first argument."""
        return cls.count

    @staticmethod
    def utility():
        """No self or cls — just a function in the class namespace."""
        return "I'm a utility"
```

### Dunder (Magic) Methods

| Method               | Triggered by                |
| -------------------- | --------------------------- |
| `__init__`           | Object creation             |
| `__str__`            | `str(obj)`, `print(obj)`   |
| `__repr__`           | `repr(obj)`, REPL display  |
| `__len__`            | `len(obj)`                  |
| `__getitem__`        | `obj[key]`                  |
| `__setitem__`        | `obj[key] = val`            |
| `__delitem__`        | `del obj[key]`              |
| `__contains__`       | `item in obj`               |
| `__iter__`           | `for item in obj`           |
| `__next__`           | `next(obj)`                 |
| `__call__`           | `obj()`                     |
| `__eq__`             | `obj == other`              |
| `__lt__`, `__gt__`   | `<`, `>`                   |
| `__add__`            | `obj + other`               |
| `__hash__`           | `hash(obj)`                 |
| `__enter__`, `__exit__` | `with obj:`              |
| `__del__`            | Object destruction (rare)   |

### Abstract Base Classes

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        pass

class Rectangle(Shape):
    def __init__(self, w, h):
        self.w, self.h = w, h

    def area(self):
        return self.w * self.h

# Shape()         # TypeError — can't instantiate abstract class
Rectangle(3, 4).area()  # 12
```

### Dataclasses (Python 3.7+)

```python
from dataclasses import dataclass, field

@dataclass
class Point:
    x: float
    y: float
    label: str = "origin"

p = Point(1.0, 2.0)
print(p)   # Point(x=1.0, y=2.0, label='origin')

# Auto-generates __init__, __repr__, __eq__, etc.

@dataclass(frozen=True)    # immutable
class FrozenPoint:
    x: float
    y: float
```

---

## 9. Exception Handling

### try / except / else / finally

```python
try:
    result = 10 / 0
except ZeroDivisionError as e:
    print(f"Error: {e}")
except (TypeError, ValueError):
    print("Type or Value error")
except Exception as e:
    print(f"Unexpected: {e}")
else:
    # Runs only if NO exception occurred
    print("Success:", result)
finally:
    # ALWAYS runs (cleanup)
    print("Done")
```

### Common Exception Hierarchy

```
BaseException
├── SystemExit
├── KeyboardInterrupt
├── GeneratorExit
└── Exception
    ├── StopIteration
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   └── OverflowError
    ├── LookupError
    │   ├── IndexError
    │   └── KeyError
    ├── AttributeError
    ├── TypeError
    ├── ValueError
    ├── FileNotFoundError
    ├── ImportError
    ├── OSError
    ├── RuntimeError
    │   └── RecursionError
    └── NameError
```

### Raising Exceptions

```python
raise ValueError("Invalid input")
raise    # re-raise current exception (inside except block)
```

### Custom Exceptions

```python
class InsufficientFundsError(Exception):
    def __init__(self, balance, amount):
        self.balance = balance
        self.amount = amount
        super().__init__(f"Cannot withdraw {amount}, balance is {balance}")

try:
    raise InsufficientFundsError(100, 200)
except InsufficientFundsError as e:
    print(e)           # Cannot withdraw 200, balance is 100
    print(e.balance)   # 100
```

### `assert`

```python
assert condition, "Error message"
# Raises AssertionError if condition is False
# Disabled when Python runs with -O flag (optimized mode)
```

---

## 10. File Handling

### Reading & Writing

```python
# Using context manager (preferred — auto-closes file)
with open("file.txt", "r") as f:
    content = f.read()          # entire file as string
    # or
    lines = f.readlines()      # list of lines
    # or
    for line in f:             # line by line (memory efficient)
        print(line.strip())

with open("file.txt", "w") as f:    # write (overwrites)
    f.write("Hello\n")

with open("file.txt", "a") as f:    # append
    f.write("World\n")
```

### File Modes

| Mode | Description               |
| ---- | ------------------------- |
| `r`  | Read (default)            |
| `w`  | Write (truncates)         |
| `a`  | Append                    |
| `x`  | Exclusive create (fails if exists) |
| `b`  | Binary mode               |
| `t`  | Text mode (default)       |
| `+`  | Read and write            |

### Working with JSON

```python
import json

# Write
data = {"name": "Alice", "age": 30}
with open("data.json", "w") as f:
    json.dump(data, f, indent=2)

# Read
with open("data.json", "r") as f:
    data = json.load(f)

# String conversion
json_str = json.dumps(data)
data = json.loads(json_str)
```

### Working with CSV

```python
import csv

# Write
with open("data.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["name", "age"])
    writer.writerow(["Alice", 30])

# Read
with open("data.csv", "r") as f:
    reader = csv.reader(f)
    for row in reader:
        print(row)

# DictReader / DictWriter
with open("data.csv", "r") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row["name"], row["age"])
```

### `os` and `pathlib` for File Operations

```python
import os
os.path.exists("file.txt")
os.path.isfile("file.txt")
os.path.isdir("dir/")
os.listdir(".")
os.makedirs("a/b/c", exist_ok=True)
os.remove("file.txt")
os.rename("old.txt", "new.txt")

from pathlib import Path   # modern, preferred
p = Path("dir/file.txt")
p.exists()
p.is_file()
p.read_text()
p.write_text("Hello")
p.parent                   # Path('dir')
p.stem                     # 'file'
p.suffix                   # '.txt'
list(Path(".").glob("*.py"))
```

---

## 11. Iterators & Generators

### Iterators

An **iterator** is any object implementing `__iter__()` and `__next__()`.

```python
class CountUp:
    def __init__(self, limit):
        self.limit = limit
        self.current = 0

    def __iter__(self):
        return self

    def __next__(self):
        if self.current >= self.limit:
            raise StopIteration
        self.current += 1
        return self.current

for num in CountUp(5):
    print(num)   # 1, 2, 3, 4, 5
```

Every iterable (list, tuple, str, dict, etc.) has an `__iter__()` method that returns an iterator.

```python
lst = [1, 2, 3]
it = iter(lst)      # calls lst.__iter__()
next(it)            # 1
next(it)            # 2
next(it)            # 3
next(it)            # StopIteration
```

### Generators

Functions that use `yield` instead of `return`. They produce values **lazily**.

```python
def fibonacci(n):
    a, b = 0, 1
    for _ in range(n):
        yield a
        a, b = b, a + b

list(fibonacci(7))   # [0, 1, 1, 2, 3, 5, 8]

gen = fibonacci(3)
next(gen)   # 0
next(gen)   # 1
next(gen)   # 1
next(gen)   # StopIteration
```

### `yield` vs `return`

| `return`                        | `yield`                              |
| ------------------------------- | ------------------------------------ |
| Terminates function             | Suspends function, resumes on `next()` |
| Returns one value               | Can yield many values                |
| Entire result in memory         | Lazy — one value at a time           |

### `yield from`

Delegates to a sub-generator.

```python
def chain(*iterables):
    for it in iterables:
        yield from it

list(chain([1,2], [3,4]))  # [1, 2, 3, 4]
```

### Generator Use Cases

- Reading large files line by line
- Infinite sequences
- Data pipelines
- Memory-efficient processing

---

## 12. Decorators

A decorator is a function that takes another function and extends its behavior **without modifying it**.

### Basic Decorator

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("Before")
        result = func(*args, **kwargs)
        print("After")
        return result
    return wrapper

@my_decorator
def say_hello():
    print("Hello!")

say_hello()
# Before
# Hello!
# After
```

`@my_decorator` is syntactic sugar for `say_hello = my_decorator(say_hello)`.

### Preserving Metadata with `functools.wraps`

```python
from functools import wraps

def my_decorator(func):
    @wraps(func)      # preserves func.__name__, __doc__, etc.
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

### Decorator with Arguments

```python
def repeat(n):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(n):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def greet():
    print("Hi!")

greet()   # prints "Hi!" three times
```

### Class-based Decorator

```python
class CountCalls:
    def __init__(self, func):
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"Call #{self.count}")
        return self.func(*args, **kwargs)

@CountCalls
def say_hi():
    print("Hi!")
```

### Common Built-in Decorators

- `@staticmethod` — no `self` or `cls`
- `@classmethod` — receives `cls`
- `@property` — getter
- `@functools.lru_cache` — memoization
- `@functools.wraps` — preserve metadata
- `@abc.abstractmethod` — abstract method

---

## 13. Lambda, Map, Filter, Reduce

### Lambda

Anonymous, single-expression function.

```python
square = lambda x: x ** 2
square(5)   # 25

add = lambda a, b: a + b
add(2, 3)   # 5

# Commonly used with sorted, map, filter
sorted(["banana", "apple", "cherry"], key=lambda s: len(s))
# ['apple', 'banana', 'cherry']
```

### `map()`

Applies a function to every item in an iterable. Returns an iterator.

```python
list(map(str, [1, 2, 3]))        # ['1', '2', '3']
list(map(lambda x: x**2, [1,2,3]))  # [1, 4, 9]

# Multiple iterables
list(map(lambda a, b: a + b, [1,2], [3,4]))  # [4, 6]
```

### `filter()`

Filters items where the function returns `True`. Returns an iterator.

```python
list(filter(lambda x: x > 0, [-1, 0, 1, 2]))  # [1, 2]
list(filter(None, [0, 1, "", "hello", [], [1]]))  # [1, 'hello', [1]] — removes falsy
```

### `reduce()`

Reduces an iterable to a single value by applying a function cumulatively.

```python
from functools import reduce

reduce(lambda a, b: a + b, [1, 2, 3, 4])    # 10
reduce(lambda a, b: a * b, [1, 2, 3, 4])    # 24
reduce(lambda a, b: a if a > b else b, [3, 1, 4, 1, 5])  # 5

# With initial value
reduce(lambda a, b: a + b, [1, 2, 3], 100)  # 106
```

---

## 14. Modules & Packages

### Importing

```python
import math
math.sqrt(16)           # 4.0

from math import sqrt, pi
sqrt(16)                # 4.0

from math import *      # import all — discouraged

import math as m
m.sqrt(16)

from collections import defaultdict as dd
```

### Creating a Module

Any `.py` file is a module.

```python
# mymodule.py
def greet(name):
    return f"Hello, {name}!"

PI = 3.14159
```

```python
# main.py
import mymodule
mymodule.greet("Alice")
```

### Packages

A package is a directory with an `__init__.py` file.

```
mypackage/
├── __init__.py
├── module_a.py
└── subpackage/
    ├── __init__.py
    └── module_b.py
```

```python
from mypackage import module_a
from mypackage.subpackage import module_b
```

### `if __name__ == "__main__":`

```python
def main():
    print("Running as script")

if __name__ == "__main__":
    main()
```

- When run directly: `__name__` is `"__main__"`
- When imported: `__name__` is the module name

### `__all__`

Controls what `from module import *` exports.

```python
# mymodule.py
__all__ = ["public_func"]

def public_func(): ...
def _private_func(): ...
```

---

## 15. `*args` & `**kwargs`

### `*args` — Variable Positional Arguments

```python
def my_sum(*args):
    return sum(args)

my_sum(1, 2, 3)     # 6
my_sum(1, 2, 3, 4)  # 10
```

`args` is a **tuple** of all extra positional arguments.

### `**kwargs` — Variable Keyword Arguments

```python
def info(**kwargs):
    for key, value in kwargs.items():
        print(f"{key}: {value}")

info(name="Alice", age=30)
# name: Alice
# age: 30
```

`kwargs` is a **dict** of all extra keyword arguments.

### Combining

```python
def func(a, b, *args, **kwargs):
    print(a, b, args, kwargs)

func(1, 2, 3, 4, x=5, y=6)
# 1 2 (3, 4) {'x': 5, 'y': 6}
```

**Order must be:** positional → `*args` → keyword → `**kwargs`

### Unpacking with `*` and `**`

```python
def add(a, b, c):
    return a + b + c

args = [1, 2, 3]
add(*args)            # 6

kwargs = {"a": 1, "b": 2, "c": 3}
add(**kwargs)         # 6

# Unpacking in assignments
first, *middle, last = [1, 2, 3, 4, 5]
# first=1, middle=[2,3,4], last=5

# Merging dicts
merged = {**dict1, **dict2}

# Merging lists
combined = [*list1, *list2]
```

---

## 16. Scope & Closures (LEGB)

### LEGB Rule

Python resolves names in this order:

1. **L**ocal — inside the current function
2. **E**nclosing — inside enclosing functions (closures)
3. **G**lobal — module-level
4. **B**uilt-in — Python's built-in names

```python
x = "global"

def outer():
    x = "enclosing"

    def inner():
        x = "local"
        print(x)     # local

    inner()

outer()
```

### `global` and `nonlocal`

```python
count = 0

def increment():
    global count      # refers to module-level count
    count += 1

def outer():
    x = 10
    def inner():
        nonlocal x    # refers to enclosing scope's x
        x += 1
    inner()
    print(x)          # 11
```

### Closures

A closure is a function that remembers variables from its enclosing scope even after that scope has finished executing.

```python
def make_multiplier(n):
    def multiplier(x):
        return x * n   # n is "closed over"
    return multiplier

double = make_multiplier(2)
triple = make_multiplier(3)

double(5)    # 10
triple(5)    # 15
```

### Late Binding Gotcha

```python
funcs = [lambda: i for i in range(5)]
[f() for f in funcs]   # [4, 4, 4, 4, 4] — all reference the same 'i'

# Fix with default argument
funcs = [lambda i=i: i for i in range(5)]
[f() for f in funcs]   # [0, 1, 2, 3, 4]
```

---

## 17. Shallow Copy vs Deep Copy

### Assignment (No Copy)

```python
a = [1, [2, 3]]
b = a           # b points to the SAME object
b[0] = 99
print(a)        # [99, [2, 3]] — a is also changed
```

### Shallow Copy

Creates a new outer object, but nested objects are still shared.

```python
import copy
a = [1, [2, 3]]

b = a.copy()            # or list(a) or a[:]
b = copy.copy(a)

b[0] = 99
print(a)    # [1, [2, 3]]  — outer not affected

b[1][0] = 99
print(a)    # [1, [99, 3]] — inner IS affected (shared reference)
```

### Deep Copy

Recursively copies everything.

```python
import copy
a = [1, [2, 3]]
b = copy.deepcopy(a)

b[1][0] = 99
print(a)    # [1, [2, 3]]  — completely independent
```

---

## 18. Python Memory Management & Garbage Collection

### How Python Manages Memory

1. **Private heap** — all objects and data structures live here.
2. **Memory manager** — handles allocation/deallocation.
3. **Object-specific allocators** — optimized for int, list, etc.

### Reference Counting

Every object has a reference count. When it drops to **0**, memory is freed immediately.

```python
import sys
a = []
sys.getrefcount(a)   # 2 (a + temporary ref from getrefcount)
b = a
sys.getrefcount(a)   # 3
del b
sys.getrefcount(a)   # 2
```

### Garbage Collector (Cyclic References)

Reference counting can't handle circular references. Python's `gc` module detects and collects these.

```python
import gc
gc.collect()           # force garbage collection
gc.get_count()         # (gen0, gen1, gen2) thresholds
gc.disable()           # disable automatic GC
gc.enable()            # re-enable
```

### Generational GC

- **Generation 0** — newly created objects (collected most frequently)
- **Generation 1** — survived one collection
- **Generation 2** — long-lived objects (collected least frequently)

---

## 19. Multithreading vs Multiprocessing & the GIL

### Global Interpreter Lock (GIL)

- A mutex in **CPython** that allows only **one thread** to execute Python bytecode at a time.
- Makes multithreading **not truly parallel** for CPU-bound tasks.
- Does **not** affect I/O-bound tasks (threads release GIL during I/O).

### Threading (I/O-bound)

```python
import threading

def download(url):
    # I/O-bound work
    ...

threads = []
for url in urls:
    t = threading.Thread(target=download, args=(url,))
    t.start()
    threads.append(t)

for t in threads:
    t.join()
```

### Multiprocessing (CPU-bound)

```python
from multiprocessing import Process, Pool

def compute(n):
    return sum(i*i for i in range(n))

# Process
p = Process(target=compute, args=(10**7,))
p.start()
p.join()

# Pool
with Pool(4) as pool:
    results = pool.map(compute, [10**6, 10**7, 10**8])
```

### `concurrent.futures`

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# Threads (I/O-bound)
with ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(download, url) for url in urls]
    results = [f.result() for f in futures]

# Processes (CPU-bound)
with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(compute, data))
```

### `asyncio` (Cooperative Concurrency)

```python
import asyncio

async def fetch(url):
    await asyncio.sleep(1)   # simulated I/O
    return f"Data from {url}"

async def main():
    tasks = [fetch(url) for url in urls]
    results = await asyncio.gather(*tasks)

asyncio.run(main())
```

### Comparison

| Feature       | Threading       | Multiprocessing   | Asyncio            |
| ------------- | --------------- | ----------------- | ------------------ |
| Best for      | I/O-bound       | CPU-bound         | I/O-bound (many)   |
| GIL impact    | Yes (limited)   | No (separate GIL) | Yes (single thread)|
| Memory        | Shared          | Separate          | Shared             |
| Overhead      | Low             | High              | Very low           |

---

## 20. Common Built-in Functions

```python
# Math
abs(-5)                  # 5
round(3.14159, 2)        # 3.14
min(1, 2, 3)             # 1
max(1, 2, 3)             # 3
sum([1, 2, 3])           # 6
pow(2, 3)                # 8
divmod(10, 3)            # (3, 1)

# Type conversion
int(), float(), str(), bool(), list(), tuple(), set(), dict(), frozenset()
bin(10)                  # '0b1010'
oct(10)                  # '0o12'
hex(255)                 # '0xff'
ord('A')                 # 65
chr(65)                  # 'A'

# Iterables
len([1,2,3])             # 3
sorted([3,1,2])          # [1, 2, 3]
reversed([1,2,3])        # iterator
enumerate(["a","b"])     # (0,'a'), (1,'b')
zip([1,2], [3,4])        # (1,3), (2,4)
map(func, iterable)
filter(func, iterable)
any([False, True, False]) # True
all([True, True, False])  # False
next(iterator)

# Object introspection
type(42)                 # <class 'int'>
isinstance(42, int)      # True
issubclass(bool, int)    # True
id(obj)                  # memory address (CPython)
dir(obj)                 # list of attributes
hasattr(obj, "name")     # True/False
getattr(obj, "name", default)
setattr(obj, "name", value)
delattr(obj, "name")
vars(obj)                # obj.__dict__
callable(obj)            # True if obj is callable

# I/O
print("hello", end="", sep=", ", file=sys.stdout)
input("Enter: ")

# Other
range(start, stop, step)
hash("hello")
help(func)
globals()                # global symbol table
locals()                 # local symbol table
exec("print(42)")        # execute string as code (dangerous!)
eval("2 + 3")            # evaluate expression (dangerous!)
compile(source, filename, mode)
```

---

## 21. Important Standard Library Modules

| Module              | Purpose                                        |
| ------------------- | ---------------------------------------------- |
| `os`                | OS interface (files, dirs, env vars)           |
| `sys`               | System-specific parameters                     |
| `pathlib`           | Object-oriented file paths                     |
| `math`              | Mathematical functions                         |
| `random`            | Random number generation                       |
| `datetime`          | Date and time manipulation                     |
| `collections`       | `Counter`, `defaultdict`, `deque`, `namedtuple`|
| `itertools`         | Efficient iterators (`chain`, `product`, etc.) |
| `functools`         | Higher-order functions (`reduce`, `lru_cache`) |
| `json`              | JSON encoding/decoding                         |
| `csv`               | CSV file reading/writing                       |
| `re`                | Regular expressions                            |
| `logging`           | Logging framework                              |
| `unittest`          | Unit testing framework                         |
| `threading`         | Thread-based parallelism                       |
| `multiprocessing`   | Process-based parallelism                      |
| `asyncio`           | Async I/O                                      |
| `subprocess`        | Spawn new processes                            |
| `argparse`          | Command-line argument parsing                  |
| `copy`              | Shallow and deep copy                          |
| `typing`            | Type hints                                     |
| `dataclasses`       | Data classes                                   |
| `enum`              | Enumerations                                   |
| `contextlib`        | Context manager utilities                      |
| `pickle`            | Object serialization                           |
| `hashlib`           | Cryptographic hashing                          |
| `sqlite3`           | SQLite database                                |

### Key `collections` Examples

```python
from collections import Counter, defaultdict, deque, OrderedDict, namedtuple

# Counter
c = Counter("mississippi")
c.most_common(2)     # [('s', 4), ('i', 4)]

# defaultdict
dd = defaultdict(int)
dd["key"] += 1       # no KeyError

# deque (double-ended queue)
dq = deque([1, 2, 3])
dq.appendleft(0)
dq.pop()
dq.popleft()
dq.rotate(1)         # [3, 0, 1, 2]

# namedtuple
Point = namedtuple("Point", "x y")
p = Point(1, 2)
```

### Key `itertools` Examples

```python
from itertools import chain, product, permutations, combinations, count, cycle, repeat, islice, groupby, accumulate

list(chain([1,2], [3,4]))                # [1, 2, 3, 4]
list(product("AB", "12"))                # [('A','1'),('A','2'),('B','1'),('B','2')]
list(permutations("ABC", 2))             # [('A','B'),('A','C'),('B','A'),...]
list(combinations("ABC", 2))             # [('A','B'),('A','C'),('B','C')]
list(islice(count(10), 5))               # [10, 11, 12, 13, 14]
list(accumulate([1,2,3,4]))              # [1, 3, 6, 10]
```

### `re` — Regular Expressions

```python
import re

re.search(r"\d+", "abc123def")       # <Match '123'>
re.match(r"\d+", "123abc")           # <Match '123'> (matches at start only)
re.findall(r"\d+", "a1b2c3")         # ['1', '2', '3']
re.sub(r"\d+", "X", "a1b2c3")        # 'aXbXcX'
re.split(r"\s+", "a b  c")           # ['a', 'b', 'c']

# Compile for reuse
pattern = re.compile(r"(\w+)@(\w+)\.(\w+)")
m = pattern.search("user@example.com")
m.group(0)   # 'user@example.com'
m.group(1)   # 'user'
m.groups()   # ('user', 'example', 'com')
```

---

## 22. Common Interview Questions & Answers

### Q1: What is the difference between a list and a tuple?

Lists are **mutable** (can be changed after creation), tuples are **immutable**. Tuples are faster, use less memory, and can be used as dictionary keys. Use tuples for fixed collections, lists for dynamic ones.

### Q2: What are `*args` and `**kwargs`?

`*args` collects extra positional arguments as a **tuple**. `**kwargs` collects extra keyword arguments as a **dict**. They allow functions to accept a variable number of arguments.

### Q3: What is the GIL?

The **Global Interpreter Lock** is a mutex in CPython that allows only one thread to execute Python bytecode at a time. It simplifies memory management but prevents true multi-threading for CPU-bound tasks. Use `multiprocessing` to bypass it.

### Q4: How is memory managed in Python?

Python uses **reference counting** as the primary mechanism and a **generational garbage collector** for cyclic references. Objects live on a private heap managed by the Python memory manager.

### Q5: Explain decorators.

A decorator is a function that wraps another function to extend its behavior. Uses the `@decorator` syntax. Common uses: logging, authentication, caching, input validation.

### Q6: What is the difference between `deepcopy` and `copy`?

`copy.copy()` creates a **shallow copy** (new outer object, shared inner objects). `copy.deepcopy()` creates a **deep copy** (recursively copies everything — fully independent).

### Q7: What are generators and why use them?

Generators are functions that use `yield` to produce values lazily. They don't store all values in memory at once. Useful for large datasets, infinite sequences, and pipelines.

### Q8: Explain list comprehension vs generator expression.

List comprehension `[x for x in range(n)]` creates the entire list in memory. Generator expression `(x for x in range(n))` produces values lazily. Use generators for large datasets.

### Q9: What is `self` in Python?

`self` refers to the current instance of a class. It's the first parameter of instance methods. Python passes it automatically when calling methods on an object.

### Q10: Difference between `__str__` and `__repr__`?

- `__str__`: Readable, user-friendly representation. Used by `print()` and `str()`.
- `__repr__`: Unambiguous, developer-friendly representation. Used by `repr()` and the REPL. Should ideally be a valid Python expression to recreate the object.

### Q11: What are context managers?

Objects that define `__enter__` and `__exit__` methods, used with the `with` statement for resource management (auto-cleanup).

```python
# Custom context manager
class Timer:
    def __enter__(self):
        import time
        self.start = time.time()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        import time
        print(f"Elapsed: {time.time() - self.start:.2f}s")
        return False  # don't suppress exceptions

with Timer():
    # ... code to time ...
    pass

# Using contextlib
from contextlib import contextmanager

@contextmanager
def timer():
    import time
    start = time.time()
    yield
    print(f"Elapsed: {time.time() - start:.2f}s")
```

### Q12: What is monkey patching?

Dynamically modifying a module or class at runtime.

```python
import math
math.pi = 3   # monkey patching — now math.pi is 3
```

Generally discouraged in production; sometimes used in testing.

### Q13: What is duck typing?

"If it walks like a duck and quacks like a duck, it's a duck." Python doesn't check types — it checks if the object has the required methods/attributes.

```python
class Duck:
    def quack(self):
        print("Quack!")

class Person:
    def quack(self):
        print("I'm quacking like a duck!")

def make_it_quack(thing):
    thing.quack()   # works with both Duck and Person

make_it_quack(Duck())
make_it_quack(Person())
```

### Q14: Explain `@staticmethod` vs `@classmethod`.

| Feature          | `@staticmethod`    | `@classmethod`         |
| ---------------- | ------------------ | ---------------------- |
| First argument   | None               | `cls` (the class)      |
| Access class     | No                 | Yes                    |
| Use case         | Utility function   | Factory methods, alternate constructors |

### Q15: What is a metaclass?

A metaclass is the "class of a class." Just as an object is an instance of a class, a class is an instance of a metaclass. The default metaclass is `type`.

```python
class MyMeta(type):
    def __new__(mcs, name, bases, namespace):
        # customize class creation
        return super().__new__(mcs, name, bases, namespace)

class MyClass(metaclass=MyMeta):
    pass
```

### Q16: How do you make a class iterable?

Implement `__iter__` (returns iterator) and `__next__` (returns next value, raises `StopIteration` when done).

### Q17: Explain `slots`.

`__slots__` restricts the attributes a class can have, saving memory by not creating a `__dict__` for each instance.

```python
class Point:
    __slots__ = ("x", "y")
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
p.z = 3   # AttributeError — not in __slots__
```

### Q18: What is the difference between `==` and `is`?

`==` checks **value equality** (calls `__eq__`). `is` checks **identity** (same object in memory — same `id()`).

### Q19: How does Python handle default mutable arguments?

Default arguments are evaluated **once** at function definition time. Mutable defaults (like lists) are shared across calls.

```python
# Bug
def append_to(item, lst=[]):
    lst.append(item)
    return lst

append_to(1)   # [1]
append_to(2)   # [1, 2] — same list!

# Fix
def append_to(item, lst=None):
    if lst is None:
        lst = []
    lst.append(item)
    return lst
```

### Q20: What is `enumerate()` and why use it?

`enumerate()` adds a counter to an iterable. Avoids manual index tracking.

```python
for i, val in enumerate(["a", "b", "c"], start=1):
    print(i, val)   # 1 a, 2 b, 3 c
```

---

## 23. Python 2 vs Python 3 Key Differences

| Feature              | Python 2                  | Python 3                   |
| -------------------- | ------------------------- | -------------------------- |
| `print`              | `print "hello"` (statement) | `print("hello")` (function) |
| Integer division     | `3/2 = 1`                | `3/2 = 1.5`               |
| Strings              | ASCII by default          | Unicode by default         |
| `range()`            | Returns list              | Returns iterator           |
| `xrange()`           | Exists                    | Removed (use `range`)      |
| `input()`            | `raw_input()`             | `input()` (always str)     |
| `dict.keys()`        | Returns list              | Returns view               |
| Exception syntax     | `except E, e:`            | `except E as e:`           |
| `__future__`         | Backport Py3 features     | Not needed                 |

> Python 2 reached **end of life on January 1, 2020**. Always use Python 3.

---

## 24. Tips & Best Practices

### PEP 8 Style Highlights

- **Indentation:** 4 spaces (never tabs)
- **Line length:** max 79 characters (72 for docstrings)
- **Naming:**
  - `snake_case` for functions, variables, methods
  - `PascalCase` for classes
  - `UPPER_SNAKE_CASE` for constants
  - `_single_leading_underscore` for internal use
- **Blank lines:** 2 between top-level definitions, 1 between methods
- **Imports:** one per line, grouped (stdlib → third-party → local)

### Pythonic Idioms

```python
# Swap variables
a, b = b, a

# Check for None
if x is None:     # ✅
if x == None:     # ❌

# Check empty collection
if not my_list:    # ✅
if len(my_list) == 0:  # ❌

# Iterate with index
for i, val in enumerate(lst):  # ✅
for i in range(len(lst)):     # ❌

# Unpack
first, *rest = [1, 2, 3, 4]

# Use context managers
with open("file.txt") as f:    # ✅
f = open("file.txt")           # ❌ (may not close on exception)

# Dictionary get with default
d.get("key", default_value)    # ✅
if "key" in d: d["key"]       # ❌ (unnecessary double lookup)

# f-strings over .format() over %
f"Hello, {name}"              # ✅ (Python 3.6+)

# Use `in` for membership
if x in my_list:               # ✅

# Chained comparison
if 0 < x < 10:                # ✅
if x > 0 and x < 10:          # ❌
```

### Common Gotchas to Remember

1. **Mutable default arguments** — use `None` as default
2. **Late binding closures** — variables in closures bind late
3. **Integer caching** — `-5` to `256` cached in CPython
4. **String interning** — not guaranteed for all strings
5. **`is` vs `==`** — identity vs equality
6. **Modifying a list while iterating** — use a copy or list comprehension
7. **Circular imports** — restructure code or use lazy imports
8. **`except:` (bare)** — catches everything including `KeyboardInterrupt`; always specify exception type

---
