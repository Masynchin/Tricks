# Rust

## The power of `reduce`

[Source](https://github.com/FrancisRussell/ferrous-actions-dev/blob/ce4ed82a21478c5b170d7f551da5dfb4ceea5055/src/cache_cargo_home.rs#L78-L87)

~~~rust
pub fn last_modified(&self) -> Option<DateTime<Utc>> {
    let mut result = None;
    for fingerprint in self.entries.values() {
        result = match (result, fingerprint.modified()) {
            (None, modified) | (modified, None) => modified,
            (Some(a), Some(b)) => Some(std::cmp::max(a, b)),
        };
    }
    result
}
~~~

Here is the summary of the types we working with:

~~~rust
struct Main {
    entries: HashMap</* K */, Fingerprint>
}

impl Main {
    fn last_modified(&self) -> Option<DateTime<Utc>> { ... }
}

trait Fingerprint {
    fn modified(&self) -> Option<DateTime<Utc>>
}
~~~

I hope you too think it is a little bit messy.
Two-level nested, pattern matching, mutability...
Hard to understand, hard to maintain.

In *N* steps we will refactor this code,
so it would be easier to work with.

For the sake of ... let's expand first branch of
the pattern matching into three separate ones.
It splits onto pair of (Some, None) and (None, Some),
and one (None, None). Let's take a look at them.

In first two cases Some is binded/bound to `modified`,
and in the third one it is None. For now, in case of
two `None`'s we will omit variable binding...

~~~rust
pub fn last_modified(&self) -> Option<DateTime<Utc>> {
    let mut result = None;
    for fingerprint in self.entries.values() {
        result = match (result, fingerprint.modified()) {
            (None,               None)               => None,
            (modified @ Some(_), None)               => modified,
            (None,               modified @ Some(_)) => modified,
            (Some(a),            Some(b))            => Some(std::cmp::max(a, b)),
        };
    }
    result
}
~~~

... but for no so long. Let's bind `modified` to the first `None`.
It won't change anything, but will allow us to make further simplifications.

~~~rust
pub fn last_modified(&self) -> Option<DateTime<Utc>> {
    let mut result = None;
    for fingerprint in self.entries.values() {
        result = match (result, fingerprint.modified()) {
            (modified @ None,    None)               => modified,
            (modified @ Some(_), None)               => modified,
            (None,               modified @ Some(_)) => modified,
            (Some(a),            Some(b))            => Some(std::cmp::max(a, b)),
        };
    }
    result
}
~~~

Now we can see that if the right hand side equals to `None`,
we always return `modified`. But that `modified` is
in fact original `result`. Let's explicitly mark it.

~~~rust
pub fn last_modified(&self) -> Option<DateTime<Utc>> {
    let mut result = None;
    for fingerprint in self.entries.values() {
        result = match (result, fingerprint.modified()) {
            (None,    None)               => result,
            (Some(_), None)               => result,
            (None,    modified @ Some(_)) => modified,
            (Some(a), Some(b))            => Some(std::cmp::max(a, b)),
        };
    }
    result
}
~~~

So, we don't change `result` if the right hand side equals to `None`.
Let's filter out this hand by using `filter_map`
on `Fingerprint::modified`, so we can compare only on `result` itself.

~~~rust
pub fn last_modified(&self) -> Option<DateTime<Utc>> {
    let mut result = None;
    for modified in self.entries.values().filter_map(Fingerprint::modified) {
        result = match result {
            None    => Some(modified),
            Some(a) => Some(std::cmp::max(a, modified)),
        };
    }
    result
}
~~~

What we came up is that our pattern matching have been reduced
from four branches to two. Quite decent already, but let's take
a better look at this match.

The first branch evaluates only once - at the very start of `for` loop.
Then, it only computes the second branch.

We could be eliminate first branch, if our result would be `Some`.
That way, our code would be even simplified to not using Option at all.

~~~rust
pub fn last_modified(&self) -> DateTime<Utc> {
    let mut result = /* ??? */;
    for modified in self.entries.values().filter_map(Fingerprint::modified) {
        result = std::cmp::max(result, modified);
    }
    result
}
~~~

And even more, we could be remove mutability, which will lead us
to remove the `result` variable at all.

~~~rust
pub fn last_modified(&self) -> DateTime<Utc> {
    self.entries
        .values()
        .filter_map(Fingerprint::modified)
        .fold(/* ??? */, std::cmp::max)
}
~~~

But we can't do this, since we have no initial element of `Fingerprint`.

So far we achieved good looking code with `fold`, which only difference
is that we have initial element. Turns out, we have basically the same
algorithm, but for a case when we can't provide initial element.
It is `reduce`, and it produces `Option`, as same as our desired code.

~~~rust
pub fn last_modified(&self) -> Option<DateTime<Utc>> {
    self.entries
        .values()
        .filter_map(Fingerprint::modified)
        .reduce(std::cmp::max)
}
~~~

Great! The only thing we can change is to use `max`,
instead of `reduce` on `std::cmp::max`.

~~~rust
pub fn last_modified(&self) -> Option<DateTime<Utc>> {
    self.entries
        .values()
        .filter_map(Fingerprint::modified)
        .max()
}
~~~

## Clear your intensions

[Source](https://github.com/CanyonTurtle/kittygame/blob/ef3ede8042f941c94eafb4bdc77e7a9c2527c521/src/lib.rs#L522-L528)

~~~rust
let mut current_found_npcs = 0;
for npc in game_state.npcs.borrow().iter() {
    current_found_npcs += match npc.following_i {
        None => 0,
        Some(_) => 1,
    }
}
~~~

This code counts *something*. This value conuts all the `npc.following_i`s
that is `Some`. How we make it more clear? By filtering out all the `None`s
and counting resulting array.

~~~rust
let current_found_npcs = game_state
    .npcs
    .borrow()
    .iter()
    .filter_map(|npc| npc.following_i)
    .count();
~~~

## Replace pattern matching with combinators

[Source](https://github.com/seanmonstar/warp/blob/1cbf029b1867505e1e6f75ae9674613ae3533710/src/filters/cookie.rs#L38-L45)

~~~rust
header::optional2().map(move |opt: Option<Cookie>| {
    let cookie = opt.and_then(|cookie| cookie.get(name).map(|x| T::from_str(x)));
    match cookie {
        Some(Ok(t)) => Some(t),
        Some(Err(_)) => None,
        None => None,
    }
})
~~~

First, let's replace lambda function with trait method:

~~~rust
header::optional2().map(move |opt: Option<Cookie>| {
    let cookie = opt.and_then(|cookie| cookie.get(name).map(T::from_str));
    match cookie {
        Some(Ok(t)) => Some(t),
        Some(Err(_)) => None,
        None => None,
    }
})
~~~

Then, let's look closer at the pattern matching. We are flatteting
`Option<Result<T, _>>` to `<Option<T>>`. It would be easier to refactor
if we would flat the `Option<Option<T>>` instead. But we can transform
`Result<T, _>` to `Option<T>` with [`Result::ok`](https://doc.rust-lang.org/std/result/enum.Result.html#method.ok).

~~~rust
header::optional2().map(move |opt: Option<Cookie>| {
    let cookie = opt
        .and_then(|cookie| cookie.get(name).map(T::from_str))
        .map(Result::ok);

    match cookie {
        Some(Some(t)) => Some(t),
        Some(None) => None,
        None => None,
    }
})
~~~

Now, we can flat out nested `Option` with [`Option::flatten`](https://doc.rust-lang.org/std/option/enum.Option.html#method.flatten),
instead of using pattern matching.

~~~rust
header::optional2().map(move |opt: Option<Cookie>| {
    opt
        .and_then(|cookie| cookie.get(name).map(T::from_str))
        .map(Result::ok)
        .flatten()
})
~~~

But we can do better. Does `map + flatten` reminds you of something?
It is exactly how `flat_map` (or Rust's `and_then`) implemented.

~~~rust
header::optional2().map(move |opt: Option<Cookie>| {
    opt
        .and_then(|cookie| cookie.get(name).map(T::from_str))
        .and_then(Result::ok)
})
~~~
