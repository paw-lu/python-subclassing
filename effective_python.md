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

```python

```

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
