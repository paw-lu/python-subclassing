---
jupyter:
  jupytext:
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

# Classes and interfaces


Inspired by chapter 5 of [Effective Python](https://effectivepython.com/).

<!-- #region tags=[] -->
## Compose classes instead of nesting many levels of built-in types
<!-- #endregion -->

Want to record the grades of a set of students.

```python
class SimpleGradebook:
    def __init__(self) -> None:
        self._grades = {}

    def add_student(self, name: str) -> None:
        self._grades[name] = []

    def report_grade(self, name: str, score: float) -> None:
        self._grades[name].append(score)

    def average_grade(self, name: str) -> float:
        grades = self._grades[name]
        return sum(grades) / len(grades)


book = SimpleGradebook()
book.add_student("Isaac Newton")
book.report_grade("Isaac Newton", 90)
book.report_grade("Isaac Newton", 95)
book.report_grade("Isaac Newton", 85)
book.average_grade("Isaac Newton")
```

Extend to keep list of grades by subject.

```python
import collections


class BySubjectGradebook:
    def __init__(self) -> None:
        self._grades = {}

    def add_student(self, name: str) -> None:
        self._grades[name] = collections.defaultdict(list)

    def report_grade(self, name: str, subject: str, grade: float) -> None:
        by_subject = self._grades[name]
        grade_list = by_subject[subject]
        grade_list.append(grade)

    def average_grade(self, name: str) -> float:
        by_subject = self._grades[name]
        total, count = 0, 0
        for grades in by_subject.values():
            total += sum(grades)
            count += len(grades)
        return total / count


book = BySubjectGradebook()
book.add_student("Albert Einstein")
book.report_grade("Albert Einstein", "Math", 75)
book.report_grade("Albert Einstein", "Math", 65)
book.report_grade("Albert Einstein", "Math", 90)
book.report_grade("Albert Einstein", "Math", 95)
book.average_grade("Albert Einstein")
```

Track the weight of each score towards the overall grade.

```python
class WeightedGradebook:
    def __init__(self) -> None:
        self._grades = {}

    def add_student(self, name: str) -> None:
        self._grades[name] = collections.defaultdict(list)

    def report_grade(
        self, name: str, subject: str, score: float, weight: float
    ) -> None:
        by_subject = self._grades[name]
        grade_list = by_subject[subject]
        grade_list.append((score, weight))

    def average_grade(self, name: str) -> float:
        by_subject = self._grades[name]
        score_sum, score_count = 0, 0
        for subject, scores in by_subject.items():
            subject_avg, total_weight = 0, 0
            for score, weight in scores:
                subject_avg += score * weight
                total_weight += weight

            score_sum += subject_avg / total_weight
            score_count += 1

        return score_sum / score_count


book = WeightedGradebook()
book.add_student("Albert Einstein")
book.report_grade("Albert Einstein", "Math", 75, 0.05)
book.report_grade("Albert Einstein", "Math", 65, 0.15)
book.report_grade("Albert Einstein", "Math", 70, 0.80)
book.report_grade("Albert Einstein", "Gym", 100, 0.40)
book.report_grade("Albert Einstein", "Gym", 85, 0.60)
print(book.average_grade("Albert Einstein"))
```

Now there are more positional arguments, and `average_grade` is more complicated.
Dictionaries should also avoid more than one level of nesting.
Similarly tuples should only go to two elements per tuple.

```python
import dataclasses


@dataclasses.dataclass
class Grade:
    score: float
    weight: float


class Subject:
    def __init__(self):
        self._grades = []

    def report_grade(self, score: float, weight: float) -> None:
        self._grades.append(Grade(score, weight))

    def average_grade(self) -> float:
        total, total_weight = 0, 0
        for grade in self._grades:
            total += grade.score * grade.weight
            total_weight += grade.weight
        return total / total_weight


class Student:
    def __init__(self) -> None:
        self._subjects = collections.defaultdict(Subject)

    def get_subject(self, name) -> Subject:
        return self._subjects[name]

    def average_grade(self) -> float:
        total, count = 0, 0
        for subject in self._subjects.values():
            total += subject.average_grade()
            count += 1
        return total / count


class Gradebook:
    def __init__(self) -> None:
        self._students = collections.defaultdict(Student)

    def get_student(self, name: str) -> Student:
        return self._students[name]


book = Gradebook()
albert = book.get_student("Albert Einstein")
math = albert.get_subject("Math")
math.report_grade(75, 0.05)
math.report_grade(65, 0.15)
math.report_grade(70, 0.80)
gym = albert.get_subject("Gym")
gym.report_grade(100, 0.40)
gym.report_grade(85, 0.60)
albert.average_grade()
```

<div class="alert alert-block alert-info">
  <b>Tip:</b>
    <li>Avoid making nested dictionaries, long tuples, or complex nestings of built-ins.</li>
    <li><code>namedtuple</code> and <code>dataclass</code> can be used to make lightweigt immutable data containers</li>
    <li>Move bookkeeping code to using multiple classes when your internal state dictionaries get complicated</li>
</div>



## Accept functions instead of classes for simple interfaces


Many Python built-in APIs allow custom behavior through a function input

```python
names = ['Socrates', 'Archimedes', 'Plato', 'Aristotle']
names.sort(key=len)
names
```

```python
def log_missing():
    print("Key added")
    return 0


current = {"green": 12, "blue": 3}
increments = [
    ("red", 5),
    ("blue", 17),
    ("orange", 9),
]
result = collections.defaultdict(log_missing, current)
print(f"Before:\t{result=}")

for key, amount in increments:
    result[key] += amount
print(f"After:\t{result=}")
```

Can also use stateful closure to count the number of missing keys.

```python
def increment_with_report(current, increments):
    added_count = 0

    def missing():
        nonlocal added_count  # Stateful closure
        added_count += 1
        return 0

    result = collections.defaultdict(missing, current)
    for key, amount in increments:
        result[key] += amount
    return result, added_count


result, count = increment_with_report(current, increments)
count
```

You can also use a class,
which is slightly less confusing than the stateful closure

```python
class CountMissing:
    def __init__(self):
        self.added = 0

    def missing(self):
        self.added += 1
        return 0


counter = CountMissing()
result = collections.defaultdict(counter.missing, current)  # Method ref
for key, amount in increments:
    result[key] += amount
counter.added
```

But then we have a class whose whole purpose is to be called in another function.
Can remedy this with `__call__`—
which allows an object to be called just like a function.
It also causes the `callable` built-int function to return `True` for such an instance.

```python
class BetterCountMissing:
    def __init__(self):
        self.added = 0

    def __call__(self):
        self.added += 1
        return 0


counter = BetterCountMissing()
result = collections.defaultdict(counter, current)  # Relies on __call__
for key, amount in increments:
    result[key] += amount
counter.added
```

<div class="alert alert-block alert-info">
  <b>Tip:</b>
    <li>Instead of defining and instantiating classes, you can use functions for simple interfaces.</li>
    <li>Functions and methods in Python are first clas—they can be used in expressions.</li>
    <li>The <code>__call__</code> special method enablesr instances of a class to be called like a function</li>
    <li>When you need a function to maintain state, defined a class that provides the <code>__call__</code> method instead of stateful closure</li>
</div>


## Use `@classmethod` polymorphism to construct objects generically


Many classes can fullfil the same interface or abstract base class.

If implementing MapReduce:

```python
import os
import threading
from os import PathLike
from typing import Generator
from typing import List


class InputData:
    def read(self):
        raise NotImplementedError


class PathInputData(InputData):
    def __init__(self, path: PathLike) -> None:
        super().__init__()
        self.path = path

    def read(self) -> str:
        with open(self.path) as f:
            return f.read()


class Worker:
    def __init__(self, input_data: PathLike) -> None:
        self.input_data = input_data

    def map(self):
        raise NotImplementedError

    def reduce(self, other):
        raise NotImplementedError


class LineCountWorker(Worker):
    def map(self) -> None:
        data = self.input_data.read()
        self.result = data.count("\n")

    def reduce(self, other: int) -> None:
        self.result += other.result


def generate_inputs(data_dir: PathLike) -> Generator[PathInputData, None, None]:
    for name in os.listdir(data_dir):
        yield PathInputData(os.path.join(data_dir, name))


def create_workers(input_list: List[PathLike]) -> List[LineCountWorker]:
    workers = []
    for input_data in input_list:
        workers.append(LineCountWorker(input_data))
    return workers


def execute(workers: List[LineCountWorker]) -> int:
    threads = [threading.Thread(target=worker.map) for worker in workers]
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()

    first, *rest = workers
    for worker in rest:
        first.reduce(worker)
    return first.result


def mapreduce(data_dir: PathLike) -> str:
    inputs = generate_inputs(data_dir)
    workers = create_workers(inputs)
    return execute(workers)
```

`mapreduce` here is not genetic.
If you want to write another `InputData` or `Worker` subclass,
also need to rewrite `generate_inputs`, `create_workers` and `mapreduce` to match.

Can use polymorphism on whole class.

```python
from typing import Dict


class GenericInputData:
    def read(self):
        raise NotImplementedError

    @classmethod
    def generate_inputs(cls, config: Dict[str, PathLike]):
        raise NotImplementedError


class PathInputData(InputData):
    def __init__(self, path: PathLike) -> None:
        super().__init__()
        self.path = path

    def read(self) -> str:
        with open(self.path) as f:
            return f.read()

    @classmethod
    def generate_inputs(
        cls, config: Dict[str, PathLike]
    ) -> Generator[PathInputData, None, None]:
        data_dir = config["data_dir"]
        for name in os.listdir(data_dir):
            yield cls(os.path.join(data_dir, name))


class GenericWorker:
    def __init__(self, input_data):
        self.input_data = input_data
        self.result = None

    def map(self):
        raise NotImplementedError

    def reduce(self, other):
        raise NotImplementedError

    @classmethod
    def create_workers(
        cls, input_class: GenericInputData, config: Dict[str, PathLike]
    ) -> List["GenericWorker"]:
        workers = []
        for input_data in input_class.generate_inputs(config):
            workers.append(cls(input_data))
        return workers


class LineCountWorker(GenericWorker):
    def map(self) -> None:
        data = self.input_data.read()
        self.result = data.count("\n")

    def reduce(self, other: int) -> None:
        self.result += other.result


def mapreduce(
    worker_class: GenericWorker,
    input_class: GenericInputData,
    config: Dict[str, PathLike],
) -> int:
    workers = worker_class.create_workers(input_class, config)
    return execute(workers)
```

<div class="alert alert-block alert-info">
  <b>Tip:</b>
    <li>Use <code>@classmethod</code> to define alternative constructors for classes</li>
    <li>Use class method polymorphism to provide generic ways to build and connect many concrete subclasses</li>
</div>


## Initialize parent classes with `super`


Simple way to initialize parent class from a child is:

```python
class MyBaseClass:
    def __init__(self, value: int) -> None:
        self.value = value


class MyChildClass(MyBaseClass):
    def __init__(self) -> None:
        MyBaseClass.__init__(self, 5)


```

Mulitiple inheritance can lead to unpredictable behavior

```python
class TimesTwo:
    def __init__(self) -> None:
        self.value *= 2


class PlusFive:
    def __init__(self) -> None:
        self.value += 5


class OneWay(MyBaseClass, TimesTwo, PlusFive):
    def __init__(self, value: int) -> None:
        MyBaseClass.__init__(self, value)
        TimesTwo.__init__(self)
        PlusFive.__init__(self)


foo = OneWay(5)
print(f"First ordering value is (5 * 2) + 5 = {foo.value=}")


class AnotherWay(MyBaseClass, PlusFive, TimesTwo):
    """Multiple inheritance example.

    Even though there is a different order of suprerclasses in the
    definition, the __init__ order is the same.
    """

    def __init__(self, value: int) -> None:
        MyBaseClass.__init__(self, value)
        TimesTwo.__init__(self)
        PlusFive.__init__(self)


bar = AnotherWay(5)
print(f"Second ordering value is {bar.value=}")


class TimesSeven(MyBaseClass):
    def __init__(self, value: int) -> None:
        MyBaseClass.__init__(self, value)
        self.value *= 7


class PlusNine(MyBaseClass):
    def __init__(self, value: int) -> None:
        MyBaseClass.__init__(self, value)
        self.value += 9


class ThisWay(TimesSeven, PlusNine):
    """Diamond inheritance example.

    Both inherited classes have the same superclass, so the common
    superclass's __init__ is run multiple times -- resetting the value.
    """

    def __init__(self, value: int) -> None:
        TimesSeven.__init__(self, value)
        PlusNine.__init__(self, value)


foo = ThisWay(5)
print(f"Should be (5 * 7) + 9 = 44, but is {foo.value=}")
```

Use `super` instead.

```python
class TimesSevenCorrect(MyBaseClass):
    def __init__(self, value: int) -> None:
        super().__init__(value)
        self.value *= 7


class PlusNineCorrect(MyBaseClass):
    def __init__(self, value: int) -> None:
        super().__init__(value)
        self.value += 9


class GoodWay(TimesSevenCorrect, PlusNineCorrect):
    def __init__(self, value: int) -> None:
        super().__init__(value)


foo = GoodWay(5)
print(f"Should be 7 * (5 + 9) = 98 and is {foo.value=}")
```

Order of operation here matches MRO ordering:

```python
[cls for cls in GoodWay.mro()]
```

`super` can be called with two parameters:

1. Type of class whose MRO you're trying to access,
2. Instance on which to access that view

You whould only do this when you need to access specific functionality of a superclass's implimentation from a child class.

```python
class ExplicitTrisect(MyBaseClass):
    def __init__(self, value: int) -> None:
        super(ExplicitTrisect, self).__init__(value)
        self.value /= 3
```

These parameters are not required for object instance initialization.

```python
class AutomaticTrisect(MyBaseClass):
    def __init__(self, value):
        """Equivalent to ExplicitTrisect."""
        super(__class__, self).__init__(value)
        self.value /= 3


class ImplicitTrisect(MyBaseClass):
    """Equivalent to AutomaticTrisect."""

    def __init__(self, value):
        super().__init__(value)
        self.value /= 3


all(
    trisect(9).value == 3
    for trisect in (ExplicitTrisect, AutomaticTrisect, ImplicitTrisect)
)
```

<div class="alert alert-block alert-info">
  <b>Tip:</b>
    <li>Python's MORO solves the problem of superclass initialization order and diamond inheritance</li>
    <li>Use <code>super</code> built-int function with zero arguments to initialize parent classes</li>
</div>


## Consider composing functionality with Mix-in classes


Better to avoid multiple inheritance.
Consider _mix-ins_ if you want some of the convenience of multiple inheritance without the headaches.
A _mix-in_ is a class that defines a subset of additional methods for its child classes.
They have no instance attributes or require an `__init__`.

```python
from typing import Any
from typing import Dict
from typing import Optional



class ToDictMixin:
    def to_dict(self):
        return self._traverse_dict(self.__dict__)

    def _traverse_dict(self, instance_dict: Dict[str, Any]):
        output = {}
        for key, value in instance_dict.items():
            output[key] = self._traverse(key, value)
        return output

    def _traverse(self, key: str, value: Any):
        if isinstance(value, ToDictMixin):
            return value.to_dict()
        elif isinstance(value, dict):
            return self._traverse_dict(value)
        elif isinstance(value, list):
            return [self._traverse(key, i) for i in value]
        elif hasattr(value, "__dict__"):
            return self._traverse_dict(value.__dict__)
        else:
            return value


class BinaryTree(ToDictMixin):
    def __init__(
        self,
        value: int,
        left: Optional["BinaryTree"] = None,
        right: Optional["BinaryTree"] = None,
    ) -> None:
        self.value = value
        self.left = left
        self.right = right


tree = BinaryTree(
    10,
    left=BinaryTree(7, right=BinaryTree(9)),
    right=BinaryTree(13, left=BinaryTree(11)),
)
tree.to_dict()
```

Can also make their generic functions pluggable so behaviors can be overrriden when needed.

```python
class BinaryTreeWithParent(BinaryTree):
    """
    Since this refers to a parent, ToDictMixin.to_dict would infinitley
    loop.
    """

    def __init__(
        self,
        value: int,
        left: Optional["BinaryTree"] = None,
        right: Optional["BinaryTree"] = None,
        parent: Optional["BinaryTree"] = None,
    ):
        super().__init__(value, left=left, right=right)
        self.parent = parent

    def _traverse(self, key, value):
        """
        Override the _traverse method to only process values that
        matter.
        """
        if isinstance(value, BinaryTreeWithParent) and key == "parent":
            return value.value  # Prevent cycles
        else:
            return super()._traverse(key, value)


root = BinaryTreeWithParent(10)
root.left = BinaryTreeWithParent(7, parent=root)
root.left.right = BinaryTreeWithParent(9, parent=root.left)
root.to_dict()
```

```python
class NamedSubTree(ToDictMixin):
    """
    Also enabled any class that has an attribute of type
    BinaryTreeWithParent to automatically work with ToDictMixin.
    """

    def __init__(self, name: str, tree_with_parent: BinaryTreeWithParent) -> None:
        self.name = name
        self.tree_with_parent = tree_with_parent


my_tree = NamedSubTree("foobar", root.left.right)
my_tree.to_dict()
```

Mix-ins can be compsed together.

```python
import json
import typing
from typing import List

T = typing.TypeVar("T")


class JsonMixin:
    @classmethod
    def from_json(cls: T, data: str) -> T:
        kwargs = json.loads(data)
        return cls(**kwargs)

    def to_json(self) -> str:
        return json.dumps(self.to_dict())


class DatacenterRack(ToDictMixin, JsonMixin):
    def __init__(
        self,
        switch: Optional[Dict[str, int]] = None,
        machines: List[Dict[str, float]] = None,
    ) -> None:
        self.switch = Switch(**switch)
        self.machines = [Machine(**kwargs) for kwargs in machines]


class Switch(ToDictMixin, JsonMixin):
    def __init__(
        self, ports: Optional[int] = None, speed: Optional[float] = None
    ) -> None:
        self.ports = ports
        self.speed = speed


class Machine(ToDictMixin, JsonMixin):
    def __init__(
        self,
        cores: Optional[int] = None,
        ram: Optional[float] = None,
        disk: Optional[float] = None,
    ) -> None:
        self.cores = cores
        self.ram = ram
        self.disk = disk


# Example 11
serialized = """{
    "switch": {"ports": 5, "speed": 1e9},
    "machines": [
        {"cores": 8, "ram": 32e9, "disk": 5e12},
        {"cores": 4, "ram": 16e9, "disk": 1e12},
        {"cores": 2, "ram": 4e9, "disk": 500e9}
    ]
}"""

deserialized = DatacenterRack.from_json(serialized)
deserialized.to_json()
```

<div class="alert alert-block alert-info">
  <b>Tip:</b>
    <li>Avoid using multiple inheritance with instance attributes and <code>__init__</code> if mix-in classes can achieve the same outcome</li>
    <li>Use pluggable behaviors at the instance level to provide per-class customization</li>
        <li>Mix-ins can include instance methods or class methods</li>
            <li>Compose mix-ins to create complex functionality from simple behaviors</li>
</div>


## Prefer public attributes over private ones

```python
class MyObject:
    def __init__(self) -> None:
        self.public_field = 5
        self.__private_field = 10
    
    def get_private_field(self) -> int:
        return self.__private_field

foo = MyObject()
foo.__private_field
```

Class methods can access private attibutes

```python
class MyOtherObject:
    def __init__(self,) -> None:
        self.__private_field = 71

    @classmethod
    def get_private_field_of_instance(cls, instance: "MyOtherObject") -> int:
        return instance.__private_field



bar = MyOtherObject()
MyOtherObject.get_private_field_of_instance(bar)
```

```python
class MyParentObject:
    def __init__(self):
        self.__private_field = 71

class MyChildObject(MyParentObject):
    def get_private_field(self):
        """Can't directly access."""
        return self.__private_field

```

```python
baz = MyChildObject()
baz.get_private_field()
```

```python
baz._MyParentObject__private_field
```

```python
baz.__dict__
```

Don't use private attributes for internal APIs.
Makes subclassing hard.
Use protencted fields instead (beggining with single underscore).
Only use it when worried about naming conflics with subclasses.

```python
class ApiClass:
    def __init__(self) -> None:
        self.__value = 5

    def get(self) -> int:
        return self.__value


class Child(ApiClass):
    def __init__(self) -> None:
        super().__init__()
        self._value = "hello"

a = Child()
print(a.get(), a._value)
```

<div class="alert alert-block alert-info">
  <b>Tips:</b>
    <li>Private attributes are not rigorously enforced</li>
   <li>Only consider using private attributes to avoid naming conflics with subclasses that are beyond your control</li>
</div>


## Inherit from `collections.abc` for custom container types

```python
from collections import abc

class BadType(abc.Sequence):
    pass
foo = BadType()
```

```python
class BinaryNode:
    def __init__(self, value, left=None, right=None) -> None:
        self.value = value
        self.left = left
        self.right = right


class IndexableNode(BinaryNode):
    def _traverse(self):
        if self.left is not None:
            yield from self.left._traverse()
        yield self
        if self.right is not None:
            yield from self.right._traverse()

    def __getitem__(self, index):
        for i, item in enumerate(self._traverse()):
            if i == index:
                return item.value
        raise IndexError(f"Index {index} is out of range")


class SequenceNode(IndexableNode):
    def __len__(self) -> None:
        for count, _ in enumerate(self._traverse(), 1):
            pass
        return count


class BetterNode(SequenceNode, abc.Sequence):
    pass


tree = BetterNode(
    10,
    left=BetterNode(5, left=BetterNode(2), right=BetterNode(6, right=BetterNode(7))),
    right=BetterNode(15, left=BetterNode(11)),
)

print(tree.index(7))
print(tree.count(10))
```

<div class="alert alert-block alert-info">
  <b>Tips:</b>
    <li>Inherit directly from Python's container types `list` `dict` for simple cases</li>
   <li>Have your custom container types inherit from the interfaces defined in <code>collections.abc</code></li>
</div>
