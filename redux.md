---
jupyter:
  jupytext:
    encoding: '# -*- coding: utf-8 -*-'
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.11.2
  kernelspec:
    display_name: Python 3
    language: python
    name: python3
---

# Sublassing in Python redux


Inspired by [Subclassing in Python Redux](https://hynek.me/articles/python-subclassing-redux/).


##  Abstract data types (interfaces)


ADTs are for tightening inferance contracts.
Mostly not needed in Python thanks to `typing.Protocol` and `abc.ABCMeta.register`.

```python
def printer(r: Reader) -> None:
    print(r.read())

printer(FooReader())
```

### Nomical subtyping


If `FooReader` didn't have `read`,
instantation would fail at runtime.
`BarReader` is _not_ verfied at runtime—it is a **virtual sublass**.
Can just use `register` as a decorator.

```python
import abc


class Reader(metaclass=abc.ABCMeta):
    @abc.abstractmethod
    def read(self) -> str:
        ...


class FooReader(Reader):
    def read(self) -> str:
        return "foo"


class BarReader:
    def read(self) -> str:
        return "bar"


Reader.register(BarReader)

assert isinstance(FooReader(), Reader)
assert isinstance(BarReader(), Reader)
```

### Structural subtyping


Classes do not need to refer to know about `Protocol` structure.
Can couple with `typing.runtime_checkable()` to automatically perform `isinstance()` checks against.

```python
from typing import Protocol
from typing import runtime_checkable

@runtime_checkable
class Reader(Protocol):
    def read(self) -> str:
        ...

class FooReader:
    def read(self) -> str:
        return "foo"

assert isinstance(FooReader(), Reader)
```

## Specialization


If we say that class B specializes class A,
we say that class B is A with additional properties.
Dog is an animal plus more.
A square is _not_ a rectangle plus more.
You cannot use a square everywhere you can use a rectangle.
In general you should be able to interact with an object as if it's an instane of its base class.

Let's represent email accounts.
They all share some data—ID in database and address.
Account type can have different attributes:

- Mailbox that stores email on the serer that needs login password.
- An account that accepts emails and only forwards them to another email address that does not


### Approach 1: Create a dedicated class for each case


The way to go if model is very simple.

```python
from typing import List

class Mailbox:
    id: "UUID"
    addr: str
    pwd: str

class Forwarder:
    id: "UUID"
    addr: str
    targets: List[str]
```

### Approach 2: Create one class, make fields optional


Avoids subclassing but will lead to a lot of conditions.

```python
import enum
from typing import List
from typing import Union


class AddrType(enum.Enum):
    MAILBOX = "mailbox"
    FORWARDER = "forwarder"


class EmailAddr:
    type: AddrType
    id: "UUID"
    addr: str

    # Only useful if type == AddrType.MAILBOX
    pwd: Union[str, None]
    # Only useful if type == AddrType.FORWARDER
    target: Union[List[str], None]
```

### Approach 3: Composition


Usually good,
but in this case `EmailAddr` and `Mailbox`/`Forwarder` are too closely related.

```python
class EmailAddr:
    id: "UUID"
    addr: str

class Mailbox:
    email: EmailAddr
    pwd: str

class Forwarder:
    email: EmailAddr
    targets: List[str]
```

### Approach 4:  Create a common base class, then specialize


This is the base case for this situation.
Whenever you have a `Mailbox`
you know you have a `pwd`.
A `Mailbox` is an `EmailAddr` plus more.

```python
class EmailAddr:
    id: "UUID"
    addr: str


class Mailbox(EmailAddr):
    pwd: str


class Forwarder(EmailAddr):
    targets: list[str]
```
