---
layout: post
title:  "The Evolution of a DSL to Tagless Final"
date:   2020-02-29 18:40:23 -0800
categories: programming
tags: scala
---

## Introduction

I wrote this article to help others who don't understand why Final Tagless is
important, and to guide in how to reason about a DSL in a way that makes Final
Tagless an appropriate solution. I was stuck in a situation when, once I was
learning about Final Tagless, I didn't understand how it should be applied to
my projects or if it even could. This article mainly aligns with the process I
went through to solve these issues.

Side notes are aimed toward readers familiar with [cats][cats] and programming
with higher kinded types.

If you are more familiar with "Final Tagless Algebra" than "Final Tagless
Language," note that I will be more commonly using the latter, even though the
former is more correct.

If you already understand what a DSL is and how to write programs using monad
composition, you can skip to
[An Overview of Final Tagless][skip-introduction].

### What is a DSL?

DSL stands for "Domain-Specific Language." While the literal definition
encompasses languages like PostScript, made for printing, and Verilog, made for
hardware, a more general and commonly used definition includes any set of
operations (Turing-completeness disregarded).

A common example of a DSL is a calculator:
{% highlight scala %}
object Calculator {

  def add(a: Int, b: Int): Int = a + b
  def subtract(a: Int, b: Int): Int = a - b

}
{% endhighlight %}

We can very easily perform arbitrary operations with our calculator:
{% highlight scala %}
Calculator.add(1, 2)
// => 3

Calculator.subtract(
  Calculator.add(2, 3),
  4
)
// => 1
{% endhighlight %}

### Monads

A Monad is any sort of data structure or container that defines these two
operations:
{% highlight scala %}
trait Monad[M[_]] {
  def pure[A](a: A): M[A]
  def flatMap[A, B](ma: M[A])(f: A => M[B]): M[B]
}
{% endhighlight %}

Various other behaviors can be derived from these definitions:
{% highlight scala %}
def map[A, B](ma: M[A])(f: A => B): M[B] = flatMap(ma)(pure compose f)
def flatten[A](mma: M[M[A]]): M[A] = flatMap(mma)(identity)
def ap[A, B](ma: M[A])(mf: M[A => B]): M[B] = flatMap(mf)(map(ma))
{% endhighlight %}

For example, `List` is a Monad:
{% highlight scala %}
def pure[A](a: A): List[A] = a :: Nil
def flatMap[A, B](la: List[A])(f: A => List[B]): List[B] =
  la match {
    case Nil => Nil
    case head :: tail => f(head) ::: flatMap(tail)(f)
  }
{% endhighlight %}

So is `Option`:
{% highlight scala %}
def pure[A](a: A): Option[A] = Some(a)
def flatMap[A, B](oa: Option[A])(f: A => Option[B]): Option[B] =
  oa match {
    case None => None
    case Some(value) => f(value)
  }
{% endhighlight %}

#### The State Monad

The State Monad describes a transformation over some state, `S`, with a result
of `A`:
{% highlight scala %}
type State[S, +A] = S => (S, A)
{% endhighlight %}

Note that State is a Monad on `A`, not on `S`!

To prove that State is a Monad, we can write definitions for `pure` and
`flatMap`:
{% highlight scala %}
implicit def stateMonad[S]: Monad[State[S, *]] = new Monad[State[S, *]] {
  def pure[A](a: A): State[S, A] =
    s => (s, a)
  def flatMap[A, B](sa: State[S, A])(f: A => State[S, B]) =
    originalS => {
      val (newS, value) = sa(originalState)
      val newState = f(value)
      newState(newS)
    }
}
{% endhighlight %}

#### Purity and Side Effects

A concept that comes up often in functional programming is that of purity,
because it not only helps compilers optimize programs but helps humans
understand what certain functions can do.

One part of purity is referential transparency. For example, with
{% highlight scala %}
def myValue: Int = 3
{% endhighlight %}
it could be said that `myValue` is referentially transparent, because it
returns the same value for the same input (similar to the concept of a
mathematical function). However, it could not be said with
{% highlight scala %}
def myValueImpure: Int = {
  Random.nextInt(6)
}
{% endhighlight %}
that `myValueImpure` is referentially transparent, because it will return
different values for the same input.

The second part of purity is whether a function causes a side effect. A side
effect is any call/function/procedure that changes or requires external state.
Examples of side effects are:
- Printing a line
- Reading a file
- Making a web request
- Calling another executable

#### The State Monad as Computation

Let's create a Monad, `IO`, that is a `State` on the `RealWorld`:
{% highlight scala %}
type IO[A] = State[RealWorld, A]
{% endhighlight %}

Here, `RealWorld` is entirely hypothetical, and we will have a hypothetical
"macro" called `impure` that turns a codeblock which returns `A` into one which
returns `IO[A]`. For example:
{% highlight scala %}
def puts(str: String): IO[Unit] = impure(println(str))
val gets: IO[String] = impure(Stdio.readLine())
{% endhighlight %}

We also have a hypothetical `IOApp` that can pass an initial `RealWorld` to any
`IO`:
{% highlight scala %}
object Echo extends IOApp {
  override def run: IO[Unit] = gets.flatMap(puts)
}
{% endhighlight %}

`puts`, `gets`, and `run` are all pure, because for the same `RealWorld`,
they'll behave the same way and output the same value!

## An Overview of Final Tagless

Final Tagless is composed of three parts:
- The `Language[Wrapper[_]]`
- The `Interpreter <: Language[α]`
- The `Program[Result]`

Let's take an example where we can increment numbers:
{% highlight scala %}
trait Language[Wrapper[_]] {
  def lift(number: Int): Wrapper[Int]
  def increment(a: Wrapper[Int]): Wrapper[Int]
}
{% endhighlight %}

This language enables two things: lifting a Scala `Int` into a wrapped `Int`,
and incrementing a wrapped `Int`. We can implement this language purely like
so:
{% highlight scala %}
type Id[A] = A // Id is short for Identity
object PureInterpreter extends Language[Id] {
  def lift(number: Int): Int = number // Id[Int] => Int
  def increment(a: Int): Int = a + 1
}
{% endhighlight %}

We can also create an interpreter that does not calculate anything, but instead
records our operations in string form:
{% highlight scala %}
type AlwaysString[A] = String
object StringInterpreter extends Language[AlwaysString] {
  def lift(number: Int): String = // AlwaysString[Int] => String
    number.toString
  def increment(a: String): String = s"inc $a"
}
{% endhighlight %}

> Side note: Any `Language` where it is possible to implement an interpreter
> with `Id` as its `Wrapper` also has an implementation for any Monad as its
> `Wrapper` via `pure` and `map`/`flatMap`/`mapN`.

Lastly, we can model programs that take arbitrary interpreters:
{% highlight scala %}
trait Program[Result] {
  def apply[Wrapper[_]](interpreter: Language[Wrapper]): Wrapper[Result]
}
{% endhighlight %}

And construct them like so:
{% highlight scala %}
object IncrementFive extends Program[Int] {
  def apply[Wrapper[_]](interpreter: Language[Wrapper]): Wrapper[Int] =
    interpreter.increment(interpreter.lift(5))
}
{% endhighlight %}

We can easily apply any interpreter to a program:
{% highlight scala %}
IncrementFive(PureInterpreter)   // => 6: Int
IncrementFive(StringInterpreter) // => "inc 5": String
{% endhighlight %}

## The Evolution

Let's write a simple calculator DSL. The first step to writing a DSL is to
handle the simplest case:
{% highlight scala %}
object Calculator {

  def add(a: Int, b: Int): Int = a + b
  def subtract(a: Int, b: Int): Int = a - b
  def multiply(a: Int, b: Int): Int = a * b
  def divide(a: Int, b: Int): Int = a / b

  def show(num: Int): String = num.toString

}
{% endhighlight %}

This calculator is great, but it can't output Lisp. Let's fix that by building
another calculator:
{% highlight scala %}
object LispCalculator {

  def add(a: String, b: String): String = s"(+ $a $b)"
  def subtract(a: String, b: String): String = s"(- $a $b)"
  def multiply(a: String, b: String): String = s"(* $a $b)"
  def divide(a: String, b: String): String = s"(/ $a $b)"

  def show(l: String): String = l

}
{% endhighlight %}

Well, we have a calculator that outputs Lisp, but its not very interchangable
with our normal calculator. It takes a lot of effort to interchange them, and
we can put non-numbers into `LispCalculator`:
{% highlight scala %}
val addTwoThree: Int    = Calculator.add(2, 3)
val addTwoThree: String = LispCalculator.add("2", "3")

val unwanted: String = LispCalculator.add("apples", "oranges")
{% endhighlight %}

### The First Step

The first problem is that calculators don't share behavior. We want to have the
ability to define calculators in a generic way:
{% highlight scala %}
// A is short for Arithmetic
// S is short for Show
trait Calculator[A, S] {

  def add(a: A, b: A): A
  def subtract(a: A, b: A): A
  def multiply(a: A, b: A): A
  def divide(a: A, b: A): A

  def show(a: A): S

}
{% endhighlight %}

We can copy over our old pure calculator, as well as our Lisp calculator:
{% highlight scala %}
object PureCalculator extends Calculator[Int, String] { /* ... */ }

object LispCalculator extends Calculator[String, String] { /* ... */ }
{% endhighlight %}

However, we run into two new problems with generic computation. It's impossible
to define a return type for `calculate` that lets the `Calculation` both
perform arithmetic as well as `show`, and its impossible for `calculate` to do
anything in the first place as it doesn't know how to make values of type `A`
or `S`:
{% highlight scala %}
trait Calculation {

  def calculate[A, S](calculator: Calculator[A, S]): ???

}
{% endhighlight scala %}

### The Second Step

The second problem is that that there's no way to define a generic computation.
We can solve this by making our calculations and calculator taking some sort of
wrapper type that handles arithmetic, `show`, and any types that may be added
in the future, as well as defining a calculator  method to lift arithmetic
values:
{% highlight scala %}
trait Calculator[Wrapper[_]] {

  def lift(value: Int): Wrapper[Int]

  def add(a: Wrapper[Int], b: Wrapper[Int]): Wrapper[Int]
  def subtract(a: Wrapper[Int], b: Wrapper[Int]): Wrapper[Int]
  def multiply(a: Wrapper[Int], b: Wrapper[Int]): Wrapper[Int]
  def divide(a: Wrapper[Int], b: Wrapper[Int]): Wrapper[Int]

  def show(a: Wrapper[Int]): Wrapper[String]

}

trait Calculation[Result] {

  def apply[Wrapper[_]](calculator: Calculator[Wrapper]): Wrapper[Result]

}
{% endhighlight %}

This is now a simple Final Tagless DSL! By having the basic goals of generic
computation and generic interpretation, we've gone from an incredibly basic DSL
to a still basic, but incredibly powerful one.

Implementing calculators and calculations is super easy, and in fact we can
still copy over most of our other code:
{% highlight scala %}
type Id[A] = A
object PureCalculator extends Calculator[Id] {

  def lift(value: Int): Id[Int] = // Id[Int] => Int
    value 

  // ...

}

type AlwaysString[A] = String
object LispCalculator extends Calculator[AlwaysString] {

  def lift(value: Int): AlwaysString[Int] = // AlwaysString[Int] => String
    value.toString

  // ...

}

object TwoPlusThree extends Calculation[Int] {

  def apply[Wrapper[_]](calculator: Calculator[Wrapper]): Wrapper[Int] =
    calculator.add(calculator.lift(2), calculator.lift(3))

}

TwoPlusThree(PureCalculator) // => 5: Int
TwoPlusThree(LispCalculator) // => "(+ 2 3)": String
{% endhighlight %}

#### Using IO in Calculator

Say we want to run calculations by doing web requests. That's a side effect, so
we need to do it with IO:
{% highlight scala %}
object WebreqCalculator extends Calculator[IO] {

  def lift(value: Int): IO[Int] = IO.pure(value)

  def add(a: IO[Int], b: IO[Int]): IO[Int] =
    a.flatMap(aVal => b.flatMap(bVal => fetch(s"example.com/add/$aVal/$bVal"))
  // etc

}
{% endhighlight %}

The beauty of using IO for this is that it keeps its properties, even when
passed through a Tagless DSL:
{% highlight scala %}
TwoPlusThree(WebreqCalculator) // => IO(...): IO[Int]
// No web requests made yet!

object RunTwoPlusThree extends IOApp {
  override def run: IO[Unit] =
    TwoPlusThree(WebreqCalculator).flatMap(puts)
    // Does web requests, prints the result
}
{% endhighlight %}

## Summary

Always start with the most basic version of your DSL. Then try to generify it
with a hodge-podge of type variables, then see if you can move that to a
wrapper style.

As an exercise, I recommend creating a DSL that handles a simple party game,
with interpreters for both evaluating the state of the party game as well as
outputting a log of the state changes. For example, [Uno][uno] might look
something along the lines of:
{% highlight scala %}
case class UnoState(
  current: Card,
  numPlayers: Int,
  players: List[UnoPlayer],
  drawDeck: List[Card]
)

trait Uno[W[_]] {

  def newGame(numPlayers: Int): W[UnoState]

}
{% endhighlight %}

> Side note: another good exercise is to take the Calculator code here and
> make a `MonadCalculator` that works for any `M: Monad`.

[cats]: https://typelevel.org/cats
[skip-introduction]: #an-overview-of-final-tagless
[uno]: https://en.wikipedia.org/wiki/Uno_(card_game)
