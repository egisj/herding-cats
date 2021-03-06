---
out: composing-monadic-functions.html
---

  [KleisliSource]: $catsBaseUrl$/core/src/main/scala/cats/data/Kleisli.scala
  [354]: https://github.com/non/cats/pull/354

### Composing monadic functions

LYAHFGG:

> When we were learning about the monad laws, we said that the `<=<` function is just like composition, only instead of working for normal functions like `a -> b`, it works for monadic functions like `a -> m b`.

In Cats there's a special wrapper for a function of type `A => F[B]` called [Kleisli][KleisliSource]:

```scala
/**
 * Represents a function `A => F[B]`.
 */
final case class Kleisli[F[_], A, B](run: A => F[B]) { self =>

  ....
}

object Kleisli extends KleisliInstances with KleisliFunctions

sealed trait KleisliFunctions {
  /** creates a [[Kleisli]] from a function */
  def function[F[_], A, B](f: A => F[B]): Kleisli[F, A, B] =
    Kleisli(f)

  ....
}
```

We can use `Kleisli.function` to construct a `Kliesli` value:

```console:new
scala> :paste
object Catnip {
  implicit class IdOp[A](val a: A) extends AnyVal {
    def some: Option[A] = Some(a)
  }
  def none[A]: Option[A] = None
}
import Catnip._
scala> import cats._, cats.std.all._
scala> import cats.data.Kleisli.function
scala> val f = function { (x: Int) => (x + 1).some }
scala> val g = function { (x: Int) => (x * 100).some }
```

We can then compose the functions using `compose`, which runs the right-hand side first:

```console
scala> import cats.syntax.flatMap._
scala> 4.some >>= (f compose g).run
```

There's also `andThen`, which runs the left-hand side first:

```console
scala> 4.some >>= (f andThen g).run
```

Both `compose` and `andThen` work like function composition
but note that they retain the monadic context.

#### lift method

Kleisli also has some interesting methods like `lift`,
which allows you to lift a monadic function into another applicative functor.
When I tried using it, I realized it's broken, so here's the fixed version [#354][354]:

```scala
  def lift[G[_]](implicit G: Applicative[G]): Kleisli[λ[α => G[F[α]]], A, B] =
    Kleisli[λ[α => G[F[α]]], A, B](a => Applicative[G].pure(run(a)))
```

Here's how we can use it:

```scala
scala> val l = f.lift[List]
l: cats.data.Kleisli[[α]List[Option[α]],Int,Int] = Kleisli(<function1>)

scala> List(1, 2, 3) >>= l.run
res0: List[Option[Int]] = List(Some(2), Some(3), Some(4))
```
