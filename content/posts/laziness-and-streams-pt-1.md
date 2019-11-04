---
title: "Laziness and Streams - Part 1"
date: 2019-10-01
---

I'm currently working my way through [Functional Programming in Scala](https://www.manning.com/books/functional-programming-in-scala). Chapter 5 focuses on non-strictness as a property of functions. I found the chapter really fascinating because this property allows one to build a lazily evaluated list. In the next post, I'll try doing the same in Python. What follows here my explanation in Scala taken from the book. I try to cover the Scala specific syntax and features so hopefully this makes sense even if you're not familiar with Scala.

First the definition of **non-strictness**:

> Non-strictness is a property of a function. To say a function is non-strict just means
that the function may choose not to evaluate one or more of its arguments. In contrast, a strict function always evaluates its arguments.

The problem we're solving:

Let's say we have an array in JavaScript and we want to transform the values and also filter them. Here we increment each number in the array and then filter to get only the even numbers.

```javascript
[1, 2, 3].map(num => num + 1).filter(num => num % 2 === 0)
```

`map` and `filter` are great because we can decompose the operations on the elements of the array into different functions. The downside is that it'll take two iterations through the array; once for map and once for filter. On small arrays we won't mind the performance hit, but on larger arrays it could be an issue. It would be best if we could use map and filter the same way without the performance cost.

Below is FP In Scala's implementation of lazy list called `Stream`.

```scala
import Stream._  // required to override Scala's stdlib Stream
trait Stream[+A] {
  // The arrow => in front of the argument type B means that
  // the argument z is non-strict
  // argument f is a function whos second argument is non-strict
  def foldRight[B](z: => B)(f: (A, => B) => B): B =
    this match {
      case Cons(h,t) => f(h(), t().foldRight(z)(f))
      case _ => z
    }

  def map[B](f: A => B): Stream[B] =
    foldRight(empty: Stream[B])((a, b) => cons(f(a), b))

  def filter(f: A => Boolean): Stream[A] =
    foldRight(empty: Stream[A])((a, b) =>
      if (f(a)) cons(a, b)
      else b)

  def toList: List[A] = {
    @annotation.tailrec
    def go(s: Stream[A], acc: List[A]): List[A] = s match {
      case Cons(h,t) => go(t(), h() :: acc)
      case _ => acc
    }
    go(this, List()).reverse
  }
}
case object Empty extends Stream[Nothing]
case class Cons[+A](h: () => A, t: () => Stream[A]) extends Stream[A]

object Stream {
  def cons[A](hd: => A, tl: => Stream[A]): Stream[A] = {
    lazy val head = hd
    lazy val tail = tl
    Cons(() => head, () => tail)
  }

  def empty[A]: Stream[A] = Empty

  def apply[A](as: A*): Stream[A] =
    if (as.isEmpty) empty
    else cons(as.head, apply(as.tail: _*))
}
```

The important Scala specific syntax here is a singleton value object called `Empty`, and a case class for `Cons`. The `Stream` companion implements `apply` so one can do `Stream(...)`.
Also traits can refer to methods on their companion object, hence inside `trait Stream` we have `empty` and `cons`.

The argument types of `cons` have the `=>` operator in front of them, which means that these arguments are not [evaluated at the point of function application, but instead at each use within the function](https://www.scala-lang.org/files/archive/spec/2.13/04-basic-declarations-and-definitions.html#by-name-parameters), hence they're not strict.

In the third line of the trace above, the expressions `1` and `Stream.apply(2, 3)` are not evaluated when we pass them to `Stream.cons`  but instead when they are used inside `Stream.cons`. You'll notice that the definition of `cons` re-assigns these arguments with `lazy val`, which is like non-strictness but for variable definitions rather than arguments. `cons` passes these lazy arguments to `Cons` wrapped in a function because case classes are not allowed have lazy values. To recap, `hd` and `tl` are non-strict arguments passed to `Stream.cons`, so they are not evaluated when `Stream.cons` is called. Then inside `Stream.cons` they are made lazy so using the arguments does not evaluate them when they are wrapped in a function. Now we have a single linked list that is constructed lazily. This is nice because you could in theory build a `Stream` of very expensive expressions but they would not be evaluated until the `Stream` was traversed.

Here's the trace of a `Stream` being constructed.

```scala

Stream(1, 2, 3)
Stream.apply(1, 2, 3)
Stream.cons(1, Stream.apply(2, 3))
Stream.cons(1, Stream.cons(2, Stream.apply(3)))
Stream.cons(1, Stream.cons(2, Stream.cons(3, Stream.apply())))
Stream.cons(1, Stream.cons(2, Stream.cons(3, Stream.Empty)))
```

Back to our original problem which was to not loop through an array multiple times when using `map` and `filter`.
The tool which we use for this is `foldRight`
```scala
def foldRight[B](z: => B)(f: (A, => B) => B): B =
  this match {
    case Cons(h,t) => f(h(), t().foldRight(z)(f))
    case _ => z
  }
```
`foldRight` travels recursively to the right along the singly linked list and calls a function `f`. If it gets to the end, it returns `z` which is a non-strict expression. In the previous iteration, the right side of `f(h(), t().foldRight(z)(f))` returns `z` so we get `f(h(), z)`, the head is evaluated and we pop back up. If we wanted to sum the elements of a Stream we could do:

```scala
Stream(1, 2, 3).foldRight(0)((a, b) => a + b)
```

This will immediately evalute to `6`, but we can use `foldRight` to implement `map` and `filter`. (In the code above, map takes `f` but I renamed it to be easier to read below.)

```scala
def map[B](mapper: A => B): Stream[B] =
  foldRight(empty: Stream[B])((a, b) => cons(mapper(a), b))
```

`map` creates a new Stream but lazily because it's built on `cons`. The expression `mapper(a)` would reference `a` which is really the head of the `Stream` that foldRight is called on but since `cons` takes its parameters by-name, it's not actually evaluated. The same happens with `b` which is the expression `t().foldRight(z)(mapper)`

In other words `f(h(), t().foldRight(z)(f))` => `cons(mapper(h()), t().foldRight(Empty)(mapper))`

Now let's look at filter:

```scala
def filter(f: A => Boolean): Stream[A] =
  foldRight(empty: Stream[A])((a, b) =>
    if (f(a)) cons(a, b)
    else b)
```

The idea with `filter` is similar to `map` but we only call `cons` if `f(a)` is true.

Again because `cons` is call by name, re-assigns the parameters with `lazy val` and wraps them in a function, it doesn't evalute its arguments when they're applied to it. The other key detail is that the function `f` passed to `foldRight` has the signature `f: (A, => B) => B` so its second argument is call by name and is not evaluated when applied.

Now let's write a trace for our original add one and take even numbers program.

```scala
// _ is shorthand for an anonymous function with one arg
Stream(1, 2, 3).map(_ + 1).filter(_ % 2 == 0)

// _ + 1 is applied to the first element and we get 2
Stream.cons(2, Stream(2, 3).map(_ + 1)).filter(_ % 2 == 0)

// _ % 2 == 0 is applied to the first element 2, it passes so the item stays
// so now map and filter are called on the tail
Stream.cons(2, Stream(2, 3).map(_ + 1).filter(_ % 2 == 0))

// _ + 1 is applied to the second element and we get 3
Stream.cons(2, Stream.cons(3, Stream(3).map(_ + 1)).filter(_ % 2 == 0))

// _ % 2 == 0 is applied to the second element 3 and it's odd so  it's gone
Stream.cons(2, Stream(3).map(_ + 1).filter(_ % 2 == 0))

// _ + 1 is applied to the second element 3 so it's 4
Stream.cons(2, Stream.cons(4, Stream.empty).filter(_ % 2 == 0))

// _ % 2 is applied to the second element and it's even so it stays
Stream.cons(2, Stream.cons(4, Stream.empty))

// the final result is equivalent to
Stream(2, 4)
```

Calling `map` and `filter` won't actually evaluate the stream because they lazily construct new streams. What you can see here is that `map` and `filter`'s functions are being applied one after another to the first element, then the second and so on. This was the aha moment for me because we no longer have to repeatedly loop through a single linked list for `map` and `filter`!
If we wanted to convert the stream to a list and evaluate everything we need add a `toList` method

```scala
def toList: List[A] = {
  @annotation.tailrec
  def go(s: Stream[A], acc: List[A]): List[A] = s match {
    case Cons(h,t) => go(t(), h() :: acc)
    case _ => acc
  }
  go(this, List()).reverse
}
```

Since `go` starts from the right, we end up with the stream values in reverse order, hence the `.reverse` at the end. If we wanted to evaluate `Stream(1, 2, 3).map(_ + 1).filter(_ % 2 == 0)` we just put a `.toList` at the end. The caveat with the whole solution is that `foldRight` as it's written is not tail recursive, so it could stack overflow.

Here's a Scalafiddle you can play with:
{{< rawhtml >}}
  <iframe height="400px" frameborder="0" style="width: 100%" src="https://embed.scalafiddle.io/embed?sfid=a3xtlBc/0&theme=dark&layout=v85"></iframe>
{{< /rawhtml >}}

Pretty cool huh? At this point I was curious about being able to do this in Python and JavaScript which I'm more familiar with. I found this [great article by Jeremy Fairbank on creating functional streams in JavaScript](https://blog.jeremyfairbank.com/javascript/functional-javascript-streams-2/)

Python is my go-to language, so in the next part I'll implement `Stream` in Python. The challenge is that Python doesn't have non-strictness or laziness the same way that Scala does.