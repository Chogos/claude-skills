# Python Type Hints Cheatsheet

## Basic types

```python
x: int = 1
y: float = 3.14
name: str = "Ada"
flag: bool = True
data: bytes = b"\x00\xff"
nothing: None = None
```

## Collections

Built-in generics (Python 3.9+). No `typing.List`, `typing.Dict`, etc.

```python
names: list[str] = ["Ada", "Grace"]
scores: dict[str, int] = {"Ada": 100}
unique: set[float] = {1.0, 2.5}
pair: tuple[str, int] = ("port", 8080)
variable_tuple: tuple[int, ...] = (1, 2, 3, 4)
nested: dict[str, list[int]] = {"primes": [2, 3, 5]}
```

### collections.abc for function parameters

Prefer abstract types in signatures — accept the broadest useful type.

```python
from collections.abc import Sequence, Mapping, Iterable, Iterator

def process(items: Sequence[str]) -> None: ...       # list, tuple, etc.
def lookup(data: Mapping[str, int]) -> None: ...      # dict, OrderedDict, etc.
def consume(stream: Iterable[bytes]) -> None: ...     # any iterable
def generate() -> Iterator[int]: ...                  # generator return
```

## Union types

```python
# Python 3.10+ syntax
value: int | str = 42
maybe: str | None = None       # replaces Optional[str]

# Function signatures
def find(key: str) -> User | None: ...
def parse(raw: str | bytes) -> dict[str, object]: ...
```

## Callable

```python
from collections.abc import Callable

# Function that takes (int, str) and returns bool
handler: Callable[[int, str], bool]

# No-arg function returning None
callback: Callable[[], None]

# Accept any callable with matching signature
def retry(fn: Callable[[], str], attempts: int = 3) -> str: ...
```

### ParamSpec — preserving signatures in decorators

```python
from typing import ParamSpec, TypeVar
from collections.abc import Callable
import functools

P = ParamSpec("P")
R = TypeVar("R")

def log_calls(fn: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(fn)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        print(f"Calling {fn.__name__}")
        return fn(*args, **kwargs)
    return wrapper

@log_calls
def greet(name: str, excited: bool = False) -> str:
    return f"Hello {name}{'!' if excited else '.'}"
```

## Generics

### TypeVar

```python
from typing import TypeVar

T = TypeVar("T")

def first(items: list[T]) -> T:
    return items[0]

# Bounded TypeVar
from typing import TypeVar

Num = TypeVar("Num", int, float)

def add(a: Num, b: Num) -> Num:
    return a + b

# Upper-bound TypeVar
from typing import TypeVar

class Animal: ...
A = TypeVar("A", bound=Animal)

def clone(animal: A) -> A: ...
```

### Generic classes

```python
from typing import TypeVar, Generic

T = TypeVar("T")

class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

    def peek(self) -> T | None:
        return self._items[-1] if self._items else None

# Usage — type is inferred
stack = Stack[int]()
stack.push(42)
val: int = stack.pop()
```

## Protocol — structural subtyping

No inheritance required. If it has the right methods, it matches.

```python
from typing import Protocol, runtime_checkable

class Writable(Protocol):
    def write(self, data: str) -> int: ...

class Closeable(Protocol):
    def close(self) -> None: ...

class ReadableCloseable(Closeable, Protocol):
    def read(self, n: int = -1) -> str: ...

# Any class with a .write(str) -> int method satisfies Writable
def save(target: Writable, content: str) -> None:
    target.write(content)

# runtime_checkable enables isinstance checks
@runtime_checkable
class Sized(Protocol):
    def __len__(self) -> int: ...

assert isinstance([1, 2], Sized)
```

## Type narrowing

### isinstance

```python
def process(value: str | int | list[str]) -> str:
    if isinstance(value, str):
        return value.upper()        # narrowed to str
    if isinstance(value, int):
        return str(value)            # narrowed to int
    return ", ".join(value)          # narrowed to list[str]
```

### TypeGuard

```python
from typing import TypeGuard

def is_string_list(val: list[object]) -> TypeGuard[list[str]]:
    return all(isinstance(x, str) for x in val)

def process(items: list[object]) -> None:
    if is_string_list(items):
        # items is now list[str]
        print(items[0].upper())
```

### assert_never — exhaustiveness checking

```python
from typing import assert_never

def handle(status: str) -> str:
    match status:
        case "active":
            return "running"
        case "paused":
            return "waiting"
        case "stopped":
            return "done"
        case _ as unreachable:
            assert_never(unreachable)  # mypy error if cases are missing
```

## Overloads

Tell the type checker that return type depends on input type.

```python
from typing import overload

@overload
def get(key: str, default: None = None) -> str | None: ...
@overload
def get(key: str, default: str) -> str: ...

def get(key: str, default: str | None = None) -> str | None:
    value = lookup(key)
    return value if value is not None else default
```

## TypedDict

Typed dictionaries — for JSON-like data where you know the shape.

```python
from typing import TypedDict, NotRequired

class UserDict(TypedDict):
    name: str
    email: str
    age: int
    bio: NotRequired[str]  # optional key

def create_user(data: UserDict) -> None:
    print(data["name"])   # type-safe key access
    # data["missing"]     # mypy error: TypedDict has no key "missing"

user: UserDict = {"name": "Ada", "email": "ada@example.com", "age": 36}
```

### Nested TypedDict

```python
class Address(TypedDict):
    street: str
    city: str
    country: str

class Company(TypedDict):
    name: str
    address: Address
    employees: list[UserDict]
```

## Literal

Restrict to specific values.

```python
from typing import Literal

Mode = Literal["r", "w", "a", "rb", "wb"]

def open_file(path: str, mode: Mode = "r") -> None: ...

# Combine with overload
@overload
def fetch(url: str, format: Literal["json"]) -> dict[str, object]: ...
@overload
def fetch(url: str, format: Literal["text"]) -> str: ...

def fetch(url: str, format: Literal["json", "text"] = "json") -> dict[str, object] | str:
    ...
```

## Final

Prevent reassignment or subclassing.

```python
from typing import Final, final

MAX_RETRIES: Final = 3
API_URL: Final[str] = "https://api.example.com"

# MAX_RETRIES = 5  # mypy error: Cannot assign to final name

@final
class Singleton:
    """Cannot be subclassed."""
    ...

class Base:
    @final
    def template_method(self) -> None:
        """Cannot be overridden in subclasses."""
        ...
```

## TypeAlias

Name complex types for readability.

```python
from typing import TypeAlias

UserId: TypeAlias = int
Headers: TypeAlias = dict[str, str]
Handler: TypeAlias = Callable[[Request], Response | None]
JsonValue: TypeAlias = str | int | float | bool | None | list["JsonValue"] | dict[str, "JsonValue"]
```

## Quick reference table

| Need | Type |
|------|------|
| Maybe null | `X \| None` |
| Multiple types | `X \| Y` |
| Any list-like | `Sequence[X]` |
| Any dict-like | `Mapping[K, V]` |
| Any iterable | `Iterable[X]` |
| Function type | `Callable[[Args], Return]` |
| Fixed shape dict | `TypedDict` |
| Exact values | `Literal["a", "b"]` |
| Duck typing | `Protocol` |
| Generic param | `TypeVar("T")` |
| Complex alias | `TypeAlias` |
| Immutable | `Final` |