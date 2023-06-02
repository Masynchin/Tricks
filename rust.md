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
