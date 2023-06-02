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
