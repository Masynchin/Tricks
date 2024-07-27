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

## Needless for-comprehension

[Source](https://github.com/augustjune/canoe/blob/0930adf20a20462f7c4564fc8132d6038720b98d/core/shared/src/test/scala/canoe/api/BotSpec.scala#L34-L38)

~~~scala
for
  updates <- bot.updates.compile.toList
  texts = updates.collect { case MessageReceived(_, m: TextMessage) => m.text }
  asr <- IO(assert(texts == messages.map(_._1)))
yield asr
~~~

Here we have for-comprehension, but it doesn't really compose `IO`s.
Look at the numerated lines:

~~~scala
1  =>  updates <- bot.updates.compile.toList
2  =>  texts = updates.collect { case MessageReceived(_, m: TextMessage) => m.text }
3  =>  asr <- IO(assert(texts == messages.map(_._1)))
~~~

On the line 2 we do variable assigning, and on the line 3 singlehandedly
wrap expression in `IO`. Do we really need for-comprehension here?

Here is the version without for-comprehension:

~~~scala
bot
  .updates.compile.toList
  .map(_.collect { case MessageReceived(_, m: TextMessage) => m.text })
  .map(texts => assert(texts == messages.map(_._1)))
~~~

Looks cleaner, but we can do even better. Consider first `map` call.
We apply `map` in order to `collect` message texts from `IO[List]`.
But we can use `collect` earlier directly on `Stream`.

~~~scala
bot
  .updates
  .collect { case MessageReceived(_, m: TextMessage) => m.text }
  .compile
  .toList
  .map(texts => assert(texts == message.map(_._1)))
~~~

## Get rid of `Applicative[F].unit`

[Source](https://github.com/augustjune/canoe/blob/0930adf20a20462f7c4564fc8132d6038720b98d/examples/src/main/scala/samples/CallbackHandling.scala#L43-L55)

~~~scala
def answerCallbacks[F[_]: Monad: TelegramClient]: Pipe[F, Update, Update] =
  _.evalTap {
    case CallbackButtonSelected(_, query) =>
      query.data match {
        case Some(cbd) =>
          for {
            _ <- query.message.traverse(_.chat.send(cbd))
            _ <- query.finish
          } yield ()
        case _ => Applicative[F].unit
      }
    case _ => Applicative[F].unit
  }
~~~

This functions instantiates `Pipe`, which is anonymous function on `Stream`.
Therefore, we can operate with all the methods that `Stream` have.

There is two `Applicative[F].unit` calls, and it is probably can be replaced
by other, more descriptive operations. Let's see if we can.

First `Applicative[F].unit` called when our input `Stream` emits not a
`CallbackButtonSelected` event. In other words, we do nothing if event
is not a callback. Can we just filter out non-callbacks from input stream?
Yes, we can do it with `collect` method.

~~~scala
def answerCallbacks[F[_]: Monad: TelegramClient]: Pipe[F, Update, Update] =
  _
    .collect { case CallbackButtonSelected(_, query) => query }
    .evalTap { query =>
      query.data match {
        case Some(cbd) =>
          for {
            _ <- query.message.traverse(_.chat.send(cbd))
            _ <- query.finish
          } yield ()
        case _ => Applicative[F].unit
      }
    }
~~~

Let's explicitly put `None` as the second arm to `Some(cbd)`.

~~~scala
def answerCallbacks[F[_]: Monad: TelegramClient]: Pipe[F, Update, Update] =
  _
    .collect { case CallbackButtonSelected(_, query) => query }
    .evalTap { query =>
      query.data match {
        case Some(cbd) =>
          for {
            _ <- query.message.traverse(_.chat.send(cbd))
            _ <- query.finish
          } yield ()
        case None => Applicative[F].unit
      }
    }
~~~

The second `Applicative[F].unit` is called when callback data is `None`.

We can't use the same trick with `collect` method to filter out `None`s,
as we have to keep both `query` and `query.data` variables. The easiest way
to keep them both is to combine them into single `Option` and only then use
`collect`. How we would do it, if query is scalar, and query.data is `Option`?
Using the `tupleLeft`, as it was used in [this trick](#stream-methods).

~~~scala
def answerCallbacks[F[_]: Monad: TelegramClient]: Pipe[F, Update, Update] =
  _
    .collect { case CallbackButtonSelected(_, query) => query }
    .map { query => query.data tupleLeft query }
    .collect { case Some(query, cbd) => (query, cbd) }
    .evalTap { (query, cbd) =>
      for {
        _ <- query.message.traverse(_.chat.send(cbd))
        _ <- query.finish
      } yield ()
    }
~~~

Also, there is method that both transforms and filters `Some`s.
It is called `mapFilter`.
 
~~~scala
def answerCallbacks[F[_]: Monad: TelegramClient]: Pipe[F, Update, Update] =
  _
    .collect { case CallbackButtonSelected(_, query) => query }
    .mapFilter { query => query.data tupleLeft query }
    .evalTap { (query, cbd) =>
      for {
        _ <- query.message.traverse(_.chat.send(cbd))
        _ <- query.finish
      } yield ()
    }
~~~

This is already pretty good, but we can replace for-comprehension inside
`evalTap` with `Apply` transforms.

~~~scala
def answerCallbacks[F[_]: Monad: TelegramClient]: Pipe[F, Update, Update] =
  _
    .collect { case CallbackButtonSelected(_, query) => query }
    .mapFilter { query => query.data tupleLeft query }
    .evalTap { (query, cbd) =>
      query.message.traverse(_.chat.send(cbd)) *> query.finish.void
    }
~~~

We now can loosen typeclasses constraints from `Monad` to `Applicative`,
since we didn't use `FlatMap` instance.

~~~scala
def answerCallbacks[F[_]: Applicative: TelegramClient]: Pipe[F, Update, Update] =
  _
    .collect { case CallbackButtonSelected(_, query) => query }
    .mapFilter { query => query.data tupleLeft query }
    .evalTap { (query, cbd) =>
      query.message.traverse(_.chat.send(cbd)) *> query.finish.void
    }
~~~

## Use laws

[Source](https://github.com/scala-steward-org/scala-steward/blob/797c6f5c5ab789b766f9b4ec32d38b33367fea70/modules/core/src/main/scala/org/scalasteward/core/persistence/KeyValueStore.scala#L30-L31)

~~~scala
final def modifyF(key: K)(f: Option[V] => F[Option[V]])(implicit F: FlatMap[F]): F[Option[V]] =
  get(key).flatMap(maybeValue => f(maybeValue).flatTap(set(key, _)))
~~~

If we abstract it a little, we ca see following pattern:

~~~scala
fa.flatMap(a => f(a).flatTap(g))
~~~

It may look familiar to those, who knows the `FlatMap` laws.
In paticular, the [associativity law](https://github.com/typelevel/cats/blob/aee37023439fddadfcd1a3c1a7600f9cbfbfe796/laws/src/main/scala/cats/laws/FlatMapLaws.scala#L36-L37):

~~~scala
fa.flatMap(a => f(a).flatMap(g)) <-> fa.flatMap(f).flatMap(g)
~~~

If this works, does the following work also?

~~~scala
fa.flatMap(a => f(a).flatTap(g)) <-> fa.flatMap(f).flatTap(g)
~~~

Let's try to prove it:

~~~scala
fa.flatMap(a => f(a).flatTap(g))
// Unfolding `flatTap` by its definition:
// fa.flatTap(f) == fa.flatMap(a => as(f(a), a))
fa.flatMap(a => f(a).flatMap(a => as(g(a), a)))
// Using `flatMapAssociativity` law
// fa.flatMap(a => f(a).flatMap(g)) <-> fa.flatMap(f).flatMap(g)
// Where `a => as(g(a), a)` would be `g` from the law
fa.flatMap(f).flatMap(a => as(g(a), a)))
// Folding `flat(a => as(g(a), a))` to `flatTap(g)`
fa.flatMap(f).flatTap(g)
~~~

<details>

<summary>Coq proofs</summary>

Thanks to Aly ([@s5bug](https://github.com/s5bug)) for Coq-based proof:

~~~coq
Require Import Coq.Program.Basics.

Class monad (F : Type -> Type) := {
  pure : forall { A }, A -> F A ;
  flatMap : forall { A B }, (A -> F B) -> (F A -> F B) ;
  flatMap_assoc : forall { A B C } (g : B -> F C) (f : A -> F B), flatMap (compose (flatMap g) f) = compose (flatMap g) (flatMap f)
}.

Definition flatTap { F } { mf : monad F } { A B } (f : A -> F B) (fa : F A) : F A :=
  flatMap (fun a => flatMap (fun _ => pure a) (f a)) fa.

Theorem flatTap_assoc : forall { F } { mf : monad F }
  { A B C } (g : B -> F C) (f : A -> F B),
  flatMap (compose (flatTap g) f) = compose (flatTap g) (flatMap f).
Proof.
  intros.
  unfold flatTap.
  rewrite -> flatMap_assoc.
  unfold compose.
  reflexivity.
Qed.
~~~

That proof requires `Monad`, because there is no `map` implementation
(which comes from unfolding `as` part of `flatTap`) for `FlatMap`.

If your `FlatMap` has `map` implementation, you can loosen that proof:

~~~coq
Require Import Coq.Program.Basics.

Class flatMapClass (F : Type -> Type) := {
  map: forall { A B }, (A -> B) -> (F A -> F B) ;
  flatMap : forall { A B }, (A -> F B) -> (F A -> F B) ;
  flatMap_assoc : forall { A B C } (g : B -> F C) (f : A -> F B), flatMap (compose (flatMap g) f) = compose (flatMap g) (flatMap f)
}.

Definition flatTap { F } { mf : flatMapClass F } { A B } (f : A -> F B) (fa : F A) : F A :=
  flatMap (fun a => map (fun _ => a) (f a)) fa.

Theorem flatTap_assoc : forall { F } { mf : flatMapClass F }
  { A B C } (g : B -> F C) (f : A -> F B),
  flatMap (compose (flatTap g) f) = compose (flatTap g) (flatMap f).
Proof.
  intros.
  unfold flatTap.
  rewrite -> flatMap_assoc.
  unfold compose.
  reflexivity.
Qed.
~~~

</details>

Hoorah, we have proved that:

~~~scala
fa.flatMap(a => f(a).flatTap(g)) <-> fa.flatMap(f).flatTap(g)
~~~

Now we have the ~audacity~ justification for the rewrite of `modifyF`:

~~~scala
final def modifyF(key: K)(f: Option[V] => F[Option[V]])(implicit F: FlatMap[F]): F[Option[V]] =
  get(key).flatMap(f).flatTap(set(key, _))
~~~
