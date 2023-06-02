# Scala

## Use `for`-comprehension

[Source](https://github.com/gvolpe/gvolpe-bot/blob/a4761e133cc71f97ea20a971a3787e02b598c410/src/main/scala/io/github/gvolpe/CoolStickers.scala#LL15C1-L22C2)

~~~scala
object CoolStickers {
  def make[F[_]: Sync]: F[CoolStickers[F]] =
    Ref.of[F, Vector[String]](Vector.empty).flatMap { ref =>
      Sync[F].delay(new scala.util.Random()).map { rnd =>
        new LiveCoolStickers[F](rnd, ref)
      }
    }
}
~~~

This code sample is just a constructor with an effect of `F`, but it is hard to tell at first sight because of nested `flatMap` and `map`. Instead, we can use [`for`-comprehension](https://docs.scala-lang.org/tour/for-comprehensions.html).

~~~scala
object CoolStickers {
  def make[F[_]: Sync]: F[CoolStickers[F]] =
    for {
      ref <- Ref.of[F, Vector[String]](Vector.empty)
      rnd <- Sync[F].delay(new scala.util.Random())
    } yield new LiveCoolStickers[F](rnd, ref)
}
~~~

Much better now. I would also show you version for Scala3, where we can omit the curly braces.

~~~scala
object CoolStickers:
  def make[F[_]: Sync]: F[CoolStickers[F]] =
    for
      ref <- Ref.of[F, Vector[String]](Vector.empty)
      rnd <- Sync[F].delay(new scala.util.Random())
    yield new LiveCoolStickers[F](rnd, ref)
~~~

## `maxOption` / `lastOption`

[Source](https://github.com/augustjune/canoe/blob/0930adf20a20462f7c4564fc8132d6038720b98d/core/shared/src/main/scala/canoe/api/sources/Polling.scala#LL31C2-L35C2)

~~~scala
private def lastId(updates: List[Update]): Option[Long] =
  updates match {
    case Nil      => None
    case nonEmpty => Some(nonEmpty.map(_.updateId).max)
  }
~~~

First, let's move the `map` from the `nonEmpty` match arm. This will not affect `Nil` arm, since it returns `None`.

~~~scala
private def lastId(updates: List[Update]): Option[Long] =
  updates.map(_.updateId) match {
    case Nil      => None
    case nonEmpty => Some(nonEmpty.max)
  }
~~~

Second, let's notice that we use `max`, but only in case of non-empty list. We should check this explicitly because `max` fails with exception when there is no elements in collection. So, `max` should be used in case of non-empty collections, which is not our case. Our case is [`maxOption`](https://dotty.epfl.ch/api/scala/collection/IterableOnceOps.html#maxOption-fffff45e), which handles the case of empty collections not by throwing exception, but rather returing `Option`.

~~~scala
private def lastId(updates: List[Update]): Option[Long] =
  updates.map(_.updateId).maxOption
~~~

Another solution provided by @ivan-klass is to use `lastOption` (another method ending with `-Option`). It works because `updates` is pre-sorted, but is context-dependent change, so it is not a shame not to notice such optimization.

~~~scala
private def lastId(updates: List[Update]): Option[Long] =
  updates.lastOption.map(_.updateId)
~~~

It is, in fact, optimization, because it doesn't allocate new list. Whereas our previous solution first uses `map`, which returns new list, and secondly `maxOption` to return `Option`, the new solution returns `Option` right after `lastOption`, and `map` applies to that `Option`, and not to a list.
