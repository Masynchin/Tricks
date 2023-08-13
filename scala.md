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

Another solution [provided](https://github.com/augustjune/canoe/pull/463#discussion_r1212093968) by [@ivan-klass](https://github.com/ivan-klass) is to use `lastOption` (another method ending with `-Option`). It works because `updates` is pre-sorted, but is context-dependent change, so it is not a shame not to notice such optimization.

~~~scala
private def lastId(updates: List[Update]): Option[Long] =
  updates.lastOption.map(_.updateId)
~~~

It is, in fact, optimization, because it doesn't allocate new list. Whereas our previous solution first uses `map`, which returns new list, and secondly `maxOption` to return `Option`, the new solution returns `Option` right after `lastOption`, and `map` applies to that `Option`, and not to a list.

## `Stream` methods

[Source](https://github.com/vpavkin/telegram-bot-fs2/blob/0940a28505842dfd0ef0310a2e83af96d377b6cd/src/main/scala/ru/pavkin/telegram/todolist/TodoListBot.scala#L31-L34)

~~~scala
private def pollCommands: Stream[F, BotCommand] = for {
    update <- api.pollUpdates(0)
    chatIdAndMessage <- Stream.emits(update.message.flatMap(a => a.text.map(a.chat.id -> _)).toSeq)
  } yield BotCommand.fromRawMessage(chatIdAndMessage._1, chatIdAndMessage._2)
~~~

Because of the usage of `filterMap`, `map`, and `toSeq`, we can figure out that `update.message` and `message.text` are of type of `Option`.

First of all, let's use `fromOption` constuctor for `Stream`, instead of `Stream.emits(_.toSeq)`.

~~~scala
private def pollCommands: Stream[F, BotCommand] = for {
    update <- api.pollUpdates(0)
    chatIdAndMessage <- Stream.fromOption[F](update.message.flatMap(a => a.text.map(a.chat.id -> _)))
  } yield BotCommand.fromRawMessage(chatIdAndMessage._1, chatIdAndMessage._2)
~~~

Then, apply the same technique to `update.message`.

~~~scala
private def pollCommands: Stream[F, BotCommand] = for {
    update <- api.pollUpdates(0)
    message <- Stream.fromOption[F](update.message)
    chatIdAndMessage <- Stream.fromOption[F](message.text.map(message.chat.id -> _))
  } yield BotCommand.fromRawMessage(chatIdAndMessage._1, chatIdAndMessage._2)
~~~

Consider `message.text.map(message.chat.id -> _)`. Here we have `message.text` which is `Option[String]`, and `message.chat.id` which is `Int`. This code creates `Option[(Int, String)]`.

Instead of `.map(_ -> _)` we can use [`tupleLeft`](https://typelevel.org/cats/nomenclature.html#functor) method, provided for Option by [cats](https://github.com/typelevel/cats).

~~~scala
private def pollCommands: Stream[F, BotCommand] = for {
    update <- api.pollUpdates(0)
    message <- Stream.fromOption[F](update.message)
    chatIdAndMessage <- Stream.fromOption[F](message.text tupleLeft message.chat.id)
  } yield BotCommand.fromRawMessage(chatIdAndMessage._1, chatIdAndMessage._2)
~~~

Next, let's use [`mapFilter`](https://github.com/typelevel/fs2/blob/3c93811b7f8d367eb23b02456cd6b18dbaec9962/core/shared/src/main/scala/fs2/Stream.scala#L5339), which would allow us to use chain of `Stream` operators, instead of `for`-comprehension.

~~~scala
private def pollCommands: Stream[F, BotCommand] =
  api
    .pollUpdates(0)
    .mapFilter(_.message)
    .mapFilter(m => m.text tupleLeft m.chat.id)
    .map(BotCommand.fromRawMessage(_, _))
~~~
