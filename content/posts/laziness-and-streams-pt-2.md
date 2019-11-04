---
title: "Laziness and Streams - Part 2"
date: 2019-11-03
---

Following up from [Laziness and Streams - Part 1](/posts/laziness-and-streams-pt-1) and inspired by [Jeremy Fairbank's function streams in JavaScript](https://blog.jeremyfairbank.com/javascript/functional-javascript-streams-2/) I wanted to try to implement a lazy Stream in Python. The problem is that Python does not have non-strict arguments like Scala does with the `=>` operator. Python does have a `lambda` keyword which you can use to define functions in a single line expression. The `lambda` keyword doesn't need to take arguments, so it can be used as a non-strictness operator hack. This adds a lot of extra syntax, but it allows us to simulate non-strictness and build the same lazy stream implementation that we had in Scala. Here is the code to do the basic Stream construction with map and filter.

```python
class Stream(object):
    @staticmethod
    def create(*args):
        if not args:
            return Stream()
        return Stream(
            lambda: args[0],
            lambda: Stream.create(*(args[1:]))
        )

    def __init__(self, head=None, tail=None):
        self.head = head
        self.tail = tail

    @property
    def empty(self):
        return self.head is None

    def __str__(self):
        value = "%s" % self.head() if not self.empty else ""
        return "Stream(%s" % value + (", %s" % self.tail() if self.tail else "") + ")"

    def fold_right(self, acc, f):
        if self.empty:
            return acc
        return f(
            self.head(),
            lambda: self.tail().fold_right(acc, f)
        )

    def map(self, f):
        return self.fold_right(
            Stream(),
            lambda a, b: Stream(lambda: f(a), b)
        )

    def filter(self, f):
        return self.fold_right(
            Stream(),
            lambda a, b: (Stream(lambda: a, b) if f(a) else b())
        )
```

We can do the same lazy mapping and filtering as we did in Scala:

```python
s = Stream.create(lambda: 1, lambda: 2, lambda: 3).map(lambda i: i + 1).filter(lambda i: i % 2 == 0)
print(s)
```

We get `Stream(2, Stream(4, Stream()))`.

If we wanted to create a stream from another iterable, we could write a  `create_from_iterator` method which lazily iterates over the data to create a Stream.

```python
@staticmethod
def create_from_iterator(iterable):
    iterator = iter(iterable)
    try:
        value = iterator.__next__()
    except StopIteration:
        return Stream()
    return Stream(
        lambda: value,
        lambda: Stream.create_from_iterator(iterator)
    )
```

No we can do:

```python
s = Stream.create_from_iterator(range(1, 4)).map(lambda i: i + 1).filter(lambda i: i % 2 == 0)
print(s)
```

The result is the same as before: `Stream(2, Stream(4, Stream()))`.
However, calling next on an iterator is not pure since it changes the state of the iterator. This side effect makes evaluating the stream more than once return different results.
If we call `print(s)` again the result is `Stream(2, Stream())`.

So using iterators to create streams is not ideal because we're forced to mutate state which leads to inconsistent results. We need a different way to create streams.

## Purely generating streams

To generate the sequence `1, 2, 3` we could write a function like:

```python
def range_stream(start, end):
    def loop(current):
        if current == end:
            return Stream()
        return Stream(
            lambda: current,
            lambda: loop(current + 1)
        )
    return loop(start)
```

Running `print(range_stream(1, 4))` will print `Stream(1, Stream(2, Stream(3, Stream())))`.

Now let's say we wanted a never ending stream of the fibonacci numbers.
We can write another recursive function.

```python
def fibonacci():
    def loop(n_minus_1, n):
        return Stream(
            lambda: n_minus_1,
            lambda: loop(n, n_minus_1 + n)
        )
    return loop(0, 1)
```

If we call `fibonacci` it will give us a Stream that never terminates. So we can't actually use any of the values right now. But if we implement a `take` function on `Stream` we can take `n` fibonacci numbers into a new `Stream` and evaluate that.

```python
def take(self, n):
    if n == 0:
        return Stream()
    return Stream(
        lambda: self.head(),
        lambda: self.tail().take(n - 1)
    )
```

Running `print(fibonacci().take(5))` gives `Stream(0, Stream(1, Stream(1, Stream(2, Stream(3, Stream())))))`.

The pattern in generating streams is that we always need a recursive function. There is a function called `unfold` that allows us to simplify generating a stream.

```python
def unfold(self, state, f):
    """
    Takes a state and a function f which is passed a state
    and returns either None or a (value, state) pair
    """
    result = f(state)
    if not result:
        return Stream()
    value, next_state = result
    return Stream(
        lambda: value,
        lambda: self.unfold(next_state, f)
    )
```

We can now implement `fibonacci` using `unfold`:
```python
def fibonacci():
    return Stream().unfold(
        (0, 1),
        lambda state: (
            state[0],
            (state[1], state[0] + state[1])
        )
    )
```

We can also refactor `take` using `unfold`:

```python
def take_unfold(self, n):
    def unfolder(state):
        remaining, stream = state
        if not remaining:
            return
        return (stream.head(), (remaining - 1, stream.tail()))
    return self.unfold((n, self), unfolder)
```

Now if we wanted to implement Python's `range` function, we could do it using `unfold`:

```python
def stream_range(start, end=None):
    if end is None:
        end = start
        start = 0

    def create_range_stream(i):
        if i == end:
            return
        return (i, i + 1)

    return Stream().unfold(start, create_range_stream)
```

To make our `Stream` data type play nice with the rest of Python where you may want to use a for loop or a comprehension we can implement `__iter__` and `__next__` in a type that wraps a Stream.

If we want to be able to use streams in for loops we can do so by implementing `__iter__` and `__next__`

```python
def __iter__(self):
    """On the Stream type
    """
    return StreamIterator(self)

class StreamIterator(object):
    def __init__(self, stream):
        self.stream = stream

    def __next__(self):
        if self.stream.empty:
            raise StopIteration

        value = self.stream.head()
        self.stream = self.stream.tail()
        return value
```

We have to implement a container class for the stream iterator because `Stream` is an immutable data structure while the iterator needs to be mutated to keep track of what value to return next.

It's handy to iterate over a stream in a for loop or comprehension.

```python
for i in stream_range(5, 10):
    print(i)

[i for i in stream_range(5, 10)]
```

Here's a python fiddle with all the code: https://pyfiddle.io/fiddle/5029d21b-34f4-4a34-929d-ce69d8e954dd/?i=true

## Conclusion

Using the property of non-strictness which we faked in Python using `lambda`, we were able to write a purely functional `Stream` data structure that can be lazily constructed or evaluated. We saw that when we used a stateful iterator in `create_from_iterator`, it broke the referential transparency of our Stream data type and returned inconsistent results. So instead we wrote a pure helper function called `unfold` which can aid in constructing streams like the fibonacci numbers.

There is a big caveat with Python that we've ignored here so far, which is that if we try to evaluate too large of a Stream in Python, we'll exceed the max call stack size and get an error. Since Python does not have tail cail optimization, it can't re-interpret recursive code into a loop the same way that Scala does. So unfortunately this makes Python a bad fit for implementing purely functional data structures. It looks like someone has [found a way to write a tail recursive decorator](https://chrispenner.ca/posts/python-tail-recursion). I might explore that in another follow up post.