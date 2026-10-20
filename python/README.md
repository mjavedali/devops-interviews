```
Explain each one of them : prioritize: collections, functions, OOP, exceptions, decorators, generators, GIL/concurrency, shallow vs deep copy, and common coding patterns.

1. Basics: variables, data types, operators, input/output 
2.  Control flow: `if/elif/else`, loops, `break`, `continue`, `pass` 
3.  Collections: lists, tuples, sets, dictionaries; comprehensionsFunctions: parameters, `*args/**kwargs`, return values, scope, lambda, recursion 
4.  Strings: slicing, formatting/f-strings, common methods, regex @property
5. `__name__ == "__main__"` 
6.  File handling: text/CSV/JSON, context managers (`with`) 
7.  Iterators & generators: `iter`, `next`, `yield`, generator expressions 
8.  Decorators and closures 
9.  Type hints: `typing`, dataclasses, Pydantic basics 
10.  Concurrency: threading vs multiprocessing vs `asyncio`
```

Below are the most important Python concepts, with interview-focused explanations and small examples.

## 1. Basics: variables, data types, operators, input/output

A variable is a name pointing to an object.

```python
name = "Javed"
age = 25
is_active = True
```

Python is dynamically typed: you do not declare the type explicitly.

Common types:

```python
count = 10           # int
price = 99.5         # float
name = "Python"      # str
active = True        # bool
nothing = None       # NoneType
```

Operators:

```python
a + b      # addition
a - b      # subtraction
a * b      # multiplication
a / b      # division → always float
a // b     # floor division
a % b      # remainder
a ** b     # power

a == b     # equality
a != b
a > b
a and b    # logical AND
a or b     # logical OR
not a
```

Input/output:

```python
name = input("Enter your name: ")  # always returns a string
age = int(input("Enter age: "))

print(f"Hello, {name}. You are {age}.")
```

---

## 2. Control flow

### `if / elif / else`

Used to make decisions.

```python
score = 75

if score >= 90:
    grade = "A"
elif score >= 60:
    grade = "B"
else:
    grade = "C"
```

### Loops

`for` iterates through an iterable.

```python
for number in [1, 2, 3]:
    print(number)

for i in range(3):
    print(i)  # 0, 1, 2
```

`while` continues until its condition becomes false.

```python
count = 0

while count < 3:
    print(count)
    count += 1
```

### `break`, `continue`, and `pass`

```python
for number in range(1, 6):
    if number == 3:
        break       # exit the loop entirely
    print(number)
```

```python
for number in range(1, 6):
    if number % 2 == 0:
        continue    # skip this iteration
    print(number)   # 1, 3, 5
```

```python
def future_feature():
    pass            # placeholder; does nothing
```

---

## 3. Collections and comprehensions

Collections store multiple values. Knowing their behavior, performance, and mutability is essential.

| Collection | Ordered | Mutable | Duplicates | Typical use |
|---|---:|---:|---:|---|
| `list` | Yes | Yes | Yes | Ordered changing data |
| `tuple` | Yes | No | Yes | Fixed records/configuration |
| `set` | No | Yes | No | Unique values, fast membership |
| `dict` | Yes* | Yes | Keys unique | Key-value data |

\* Dictionaries preserve insertion order in modern Python.

### Lists

```python
users = ["Ali", "Sara", "John"]
users.append("Meera")
users[0] = "Javed"

print(users[1])       # Sara
print(users[-1])      # Meera
```

Useful methods:

```python
users.append("A")
users.extend(["B", "C"])
users.remove("A")
users.pop()           # removes and returns last item
users.sort()
```

Time complexity to remember:

- Index access: `O(1)`
- Append at end: usually `O(1)`
- Search with `in`: `O(n)`
- Insert/remove in middle: `O(n)`

### Tuples

Tuples are immutable.

```python
point = (10, 20)
x, y = point
```

Use them when values should not change, such as coordinates or function results.

```python
def get_user():
    return "Javed", 25

name, age = get_user()
```

### Sets

Sets store unique values.

```python
skills = {"Python", "Docker", "Kubernetes"}
skills.add("Terraform")
skills.add("Python")  # ignored; already present
```

Set operations:

```python
a = {1, 2, 3}
b = {3, 4, 5}

a | b   # union: {1, 2, 3, 4, 5}
a & b   # intersection: {3}
a - b   # difference: {1, 2}
```

Membership is generally `O(1)`:

```python
if "Python" in skills:
    print("Found")
```

### Dictionaries

Dictionaries map unique keys to values.

```python
user = {
    "name": "Javed",
    "age": 25,
    "skills": ["Python", "AWS"]
}

print(user["name"])
print(user.get("city", "Unknown"))
```

```python
user["age"] = 26
user["city"] = "Bengaluru"

for key, value in user.items():
    print(key, value)
```

Dictionary lookup is generally `O(1)`.

### Comprehensions

A concise way to create collections.

```python
numbers = [1, 2, 3, 4, 5]

squares = [n * n for n in numbers]
even_squares = [n * n for n in numbers if n % 2 == 0]

word_lengths = {word: len(word) for word in ["python", "api"]}
unique_lengths = {len(word) for word in ["cat", "dog", "python"]}
```

Avoid overly complex comprehensions; use a normal loop if readability suffers.

---

## 4. Functions

Functions group reusable logic.

```python
def greet(name):
    return f"Hello, {name}"

message = greet("Javed")
```

### Parameters and arguments

```python
def create_user(name, age=18, active=True):
    return {"name": name, "age": age, "active": active}

create_user("Ali")
create_user(name="Sara", age=25)
```

- Parameter: variable in function definition.
- Argument: value passed while calling the function.
- Default parameter: used when no argument is supplied.

### `*args` and `**kwargs`

`*args` collects extra positional arguments as a tuple.

```python
def total(*numbers):
    return sum(numbers)

total(10, 20, 30)
```

`**kwargs` collects extra keyword arguments as a dictionary.

```python
def show_profile(**details):
    return details

show_profile(name="Javed", role="Developer")
```

You can also unpack collections:

```python
values = [1, 2, 3]
print(*values)

settings = {"timeout": 30, "retries": 3}
connect(**settings)
```

### Return values

A function returns `None` if it has no explicit `return`.

```python
def add(a, b):
    return a + b
```

Returning multiple values is actually returning one tuple.

```python
def divide(a, b):
    return a // b, a % b

quotient, remainder = divide(10, 3)
```

### Scope: local, global, nonlocal

```python
country = "India"  # global

def show_country():
    city = "Delhi"  # local
    print(country, city)
```

Avoid changing globals unless necessary.

```python
count = 0

def increment():
    global count
    count += 1
```

`nonlocal` changes a variable in the nearest enclosing function scope.

```python
def counter():
    value = 0

    def increment():
        nonlocal value
        value += 1
        return value

    return increment
```

### Lambda functions

A small anonymous function, usually for simple one-expression operations.

```python
square = lambda n: n * n
print(square(5))
```

Commonly used with `sorted`, `map`, and `filter`.

```python
users = [{"name": "Ali", "age": 30}, {"name": "Sara", "age": 25}]

sorted_users = sorted(users, key=lambda user: user["age"])
```

### Recursion

A function calls itself. It must have a base case.

```python
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)
```

Use recursion for trees, nested data, DFS, and divide-and-conquer. For very deep recursion, prefer iterative solutions because Python has a recursion limit.

---

## 5. OOP and `@property`

Object-Oriented Programming organizes code using classes and objects.

```python
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

    def greet(self):
        return f"Hello, {self.name}"

user = User("Javed", "javed@example.com")
print(user.greet())
```

- Class: blueprint.
- Object/instance: created from a class.
- `self`: current object instance.
- `__init__`: constructor called when the object is created.

### Encapsulation

Keep internal state controlled through methods or properties.

```python
class BankAccount:
    def __init__(self, balance):
        self._balance = balance  # convention: internal attribute

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("Amount must be positive")
        self._balance += amount
```

Python does not enforce true private fields in the same way as Java. A single underscore (`_balance`) is a convention meaning “internal use.”

### Inheritance

A child class can reuse and extend a parent class.

```python
class Employee:
    def work(self):
        return "Working"

class Developer(Employee):
    def work(self):
        return "Writing code"

developer = Developer()
print(developer.work())
```

### Polymorphism

Different objects respond to the same method call differently.

```python
class Dog:
    def sound(self):
        return "Bark"

class Cat:
    def sound(self):
        return "Meow"

for animal in [Dog(), Cat()]:
    print(animal.sound())
```

### `@property`

`@property` lets you access a method like an attribute. It is useful for validation and computed fields while preserving a clean API.

```python
class User:
    def __init__(self, email):
        self.email = email

    @property
    def domain(self):
        return self.email.split("@")[1]

user = User("javed@example.com")
print(user.domain)  # example.com, not user.domain()
```

Property setter example:

```python
class Product:
    def __init__(self, price):
        self.price = price

    @property
    def price(self):
        return self._price

    @price.setter
    def price(self, value):
        if value < 0:
            raise ValueError("Price cannot be negative")
        self._price = value
```

---

## 6. Strings: slicing, formatting, methods, regex

Strings are immutable sequences of characters.

```python
text = "Python"
```

### Slicing

```python
text[0]      # P
text[-1]     # n
text[0:3]    # Pyt
text[:3]     # Pyt
text[3:]     # hon
text[::-1]   # nohtyP
```

### F-strings

The cleanest way to format strings.

```python
name = "Javed"
score = 92.456

message = f"{name} scored {score:.2f}%"
```

### Common methods

```python
text = "  Hello Python  "

text.strip()                 # "Hello Python"
text.lower()                 # lowercase
text.upper()                 # uppercase
text.replace("Python", "AI")
text.split()                 # ["Hello", "Python"]
"-".join(["2026", "08", "26"])
text.startswith("  He")
text.find("Python")          # index, or -1 if not found
```

### Regex

Regular expressions match text patterns.

```python
import re

email = "javed@example.com"
is_valid = re.fullmatch(r"[\w.-]+@[\w.-]+\.\w+", email)

text = "Order IDs: 101, 202, 303"
numbers = re.findall(r"\d+", text)
```

Use regex for pattern matching, not simple tasks that built-in string methods handle clearly.

---

## 7. `__name__ == "__main__"`

Every Python file has a special `__name__` variable.

- If run directly, `__name__` is `"__main__"`.
- If imported, `__name__` is the module name.

```python
def main():
    print("Application started")

if __name__ == "__main__":
    main()
```

This allows a file to be both reusable as an importable module and executable as a script.

---

## 8. File handling: text, CSV, JSON, and `with`

Use `with` for file operations. It automatically closes the file even if an error occurs.

### Text file

```python
with open("notes.txt", "w", encoding="utf-8") as file:
    file.write("Hello\n")

with open("notes.txt", "r", encoding="utf-8") as file:
    content = file.read()
```

Modes:

- `"r"`: read
- `"w"`: write; overwrites existing file
- `"a"`: append
- `"x"`: create only; fails if file exists
- `"b"`: binary mode

### CSV

```python
import csv

with open("users.csv", newline="", encoding="utf-8") as file:
    reader = csv.DictReader(file)
    for row in reader:
        print(row["name"])
```

Writing:

```python
with open("users.csv", "w", newline="", encoding="utf-8") as file:
    writer = csv.DictWriter(file, fieldnames=["name", "age"])
    writer.writeheader()
    writer.writerow({"name": "Javed", "age": 25})
```

### JSON

JSON maps naturally to Python dictionaries and lists.

```python
import json

data = {"name": "Javed", "skills": ["Python", "Docker"]}

with open("user.json", "w", encoding="utf-8") as file:
    json.dump(data, file, indent=2)

with open("user.json", encoding="utf-8") as file:
    loaded_data = json.load(file)
```

---

## 9. Iterators and generators

An iterable is something you can loop over: list, tuple, string, dictionary, file, etc.

An iterator produces one item at a time.

```python
numbers = [10, 20, 30]

iterator = iter(numbers)
print(next(iterator))  # 10
print(next(iterator))  # 20
```

When there are no remaining values, `next()` raises `StopIteration`.

### Generators

Generators create values lazily, one at a time. They save memory for large sequences.

```python
def count_up_to(limit):
    current = 1
    while current <= limit:
        yield current
        current += 1

for number in count_up_to(3):
    print(number)
```

`yield` pauses the function, preserves its state, and resumes on the next iteration.

A normal function returns once:

```python
def normal_function():
    return 1
```

A generator yields repeatedly:

```python
def generator_function():
    yield 1
    yield 2
```

### Generator expressions

Like list comprehensions, but lazy.

```python
squares = (n * n for n in range(1_000_000))
```

Compare:

```python
list_values = [n * n for n in range(1_000_000)]  # stores all values
generator_values = (n * n for n in range(1_000_000))  # lazy
```

Use generators for streaming files, API pages, log processing, and large datasets.

---

## 10. Decorators and closures

### Closure

A closure is an inner function that remembers variables from its enclosing function, even after the outer function finishes.

```python
def make_multiplier(factor):
    def multiply(number):
        return number * factor
    return multiply

double = make_multiplier(2)
print(double(5))  # 10
```

`multiply()` remembers `factor`.

### Decorator

A decorator wraps another function to add reusable behavior without modifying that function’s core code.

```python
from functools import wraps

def log_call(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        result = func(*args, **kwargs)
        print("Finished")
        return result
    return wrapper

@log_call
def add(a, b):
    return a + b

print(add(2, 3))
```

`@log_call` is equivalent to:

```python
add = log_call(add)
```

Common decorator use cases:

- Logging
- Authentication/authorization
- Caching
- Measuring execution time
- Input validation
- Retry behavior

Built-in example:

```python
from functools import lru_cache

@lru_cache
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

---

## 11. Type hints, dataclasses, and Pydantic

### Type hints

Type hints document expected types and help tools such as IDEs, linters, and `mypy`. Python does not enforce them by default at runtime.

```python
def add(a: int, b: int) -> int:
    return a + b

def find_user(user_id: int) -> dict[str, str] | None:
    return None
```

Useful types:

```python
from typing import Any, Callable, Iterable, Optional

names: list[str] = ["Ali", "Sara"]
scores: dict[str, int] = {"Ali": 90}
```

### Dataclasses

Dataclasses reduce boilerplate for classes used mainly to hold data.

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
    active: bool = True

user = User("Javed", 25)
```

It automatically creates methods such as `__init__`, `__repr__`, and equality comparison.

```python
User("Javed", 25) == User("Javed", 25)  # True
```

### Pydantic basics

Pydantic validates and parses data at runtime. It is widely used with FastAPI.

```python
from pydantic import BaseModel, EmailStr

class UserInput(BaseModel):
    name: str
    age: int
    email: EmailStr

user = UserInput(
    name="Javed",
    age="25",  # may be converted to int depending on validation mode
    email="javed@example.com"
)
```

Use it when data comes from external sources: APIs, JSON, forms, environment variables, or configuration.

---

## 12. Concurrency: threading, multiprocessing, and `asyncio`

Concurrency means making progress on more than one task during the same period. It does not always mean tasks run in parallel on multiple CPU cores.

### The GIL

CPython, the standard Python implementation, has a Global Interpreter Lock (GIL). It generally allows only one thread to execute Python bytecode at a time.

Effect:

- Threads are useful for I/O-bound work.
- Threads usually do not speed up CPU-bound pure-Python work.
- Multiprocessing can use multiple CPU cores for CPU-bound work.

### Threading: best for I/O-bound work

Use threads when tasks wait on network responses, files, databases, or external services.

```python
from concurrent.futures import ThreadPoolExecutor
import requests

def fetch(url):
    return requests.get(url, timeout=10).status_code

urls = ["https://example.com", "https://example.org"]

with ThreadPoolExecutor(max_workers=5) as executor:
    statuses = list(executor.map(fetch, urls))
```

Examples: API calls, downloading files, database/network requests.

### Multiprocessing: best for CPU-bound work

Use processes for computation-heavy work because each process has its own Python interpreter and GIL.

```python
from concurrent.futures import ProcessPoolExecutor

def square(number):
    return number * number

with ProcessPoolExecutor() as executor:
    results = list(executor.map(square, range(10)))
```

Examples: image processing, large calculations, data transformations, CPU-intensive algorithms.

Important: process tasks and arguments should generally be serializable (picklable).

### `asyncio`: best for high-volume async I/O

`asyncio` uses an event loop. A single thread can efficiently manage many waiting tasks.

```python
import asyncio

async def fetch_data(name):
    await asyncio.sleep(1)  # simulates network I/O
    return f"Data from {name}"

async def main():
    results = await asyncio.gather(
        fetch_data("service-a"),
        fetch_data("service-b"),
    )
    print(results)

asyncio.run(main())
```

Important keywords:

- `async def`: declares a coroutine.
- `await`: yields control while waiting.
- `asyncio.gather()`: runs independent coroutines concurrently.

Use `asyncio` with async-compatible libraries, such as `httpx`, `aiohttp`, or async database drivers. Calling blocking code inside `async def` can block the entire event loop.

| Approach | Best for | Uses multiple CPU cores? | Example |
|---|---|---:|---|
| Threading | I/O-bound tasks | Usually no | HTTP requests, file I/O |
| Multiprocessing | CPU-bound tasks | Yes | Image/data processing |
| `asyncio` | Many I/O-bound tasks | No | Thousands of API/socket tasks |

---

## 13. Shallow copy vs deep copy

This is a very common interview question.

Assignment does not copy an object:

```python
a = [1, 2, 3]
b = a

b.append(4)
print(a)  # [1, 2, 3, 4]
```

Both variables refer to the same list.

### Shallow copy

Copies only the outer object. Nested objects are still shared.

```python
import copy

original = [[1, 2], [3, 4]]
shallow = copy.copy(original)

shallow[0].append(99)

print(original)  # [[1, 2, 99], [3, 4]]
```

Ways to make shallow copies:

```python
copied_list = original.copy()
copied_list = original[:]
copied_dict = original_dict.copy()
```

### Deep copy

Copies the outer object and recursively copies nested objects.

```python
deep = copy.deepcopy(original)
deep[0].append(100)

print(original)  # unchanged
print(deep)
```

Use deep copy only when needed; it can be expensive for large objects.

---

## 14. Common coding patterns

### Frequency counter

Use a dictionary or `Counter`.

```python
from collections import Counter

text = "banana"
counts = Counter(text)
print(counts["a"])  # 3
```

### Two pointers

Useful for sorted arrays, reversing arrays, and pair-sum problems.

```python
def has_pair_with_sum(numbers, target):
    left, right = 0, len(numbers) - 1

    while left < right:
        total = numbers[left] + numbers[right]

        if total == target:
            return True
        if total < target:
            left += 1
        else:
            right -= 1

    return False
```

### Sliding window

Useful for subarrays/substrings.

```python
def max_sum_k(numbers, k):
    window_sum = sum(numbers[:k])
    best = window_sum

    for i in range(k, len(numbers)):
        window_sum += numbers[i] - numbers[i - k]
        best = max(best, window_sum)

    return best
```

### Stack

Use a list for Last-In-First-Out operations.

```python
stack = []
stack.append("A")
stack.append("B")
last = stack.pop()  # B
```

Common uses: parentheses validation, undo, DFS.

### Queue

Use `deque`, not a list, for efficient removal from the left.

```python
from collections import deque

queue = deque(["task-1"])
queue.append("task-2")
current = queue.popleft()
```

### `defaultdict`

Avoid repetitive “does key exist?” checks.

```python
from collections import defaultdict

groups = defaultdict(list)

for name, team in [("Ali", "Dev"), ("Sara", "QA")]:
    groups[team].append(name)
```

### `enumerate` and `zip`

```python
for index, value in enumerate(["a", "b"]):
    print(index, value)

for name, score in zip(["Ali", "Sara"], [90, 95]):
    print(name, score)
```

### `any` and `all`

```python
any(n > 10 for n in [2, 5, 15])  # True
all(n > 0 for n in [2, 5, 15])   # True
```

### Safe dictionary access

```python
user = {"name": "Javed"}

city = user.get("city", "Unknown")
```

### Context managers

Use `with` whenever a resource must be cleaned up.

```python
with open("log.txt") as file:
    data = file.read()
```

The key interview takeaway: select the right data structure first—`set`/`dict` for fast lookup, `deque` for queues, list for ordered indexing, and generators for memory-efficient processing.