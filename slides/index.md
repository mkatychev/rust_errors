---
marp: true
theme: default
style: @import url('https://unpkg.com/tailwindcss@^2/dist/utilities.min.css');
---


# Rust Errors:
Seeing is Observing

![bg right width:30em](https://www.shuttle.rs//images/blog/ferris-error-handling.png)

---

## What is an error?
* seriously... what is it?


* [Wikipedia](https://en.wikipedia.org/wiki/Error#cite_note-:0-1): An error (from the Latin _errāre_, meaning 'to wander') is an inaccurate or incorrect action, thought, or judgement.[1]
* Biology: a mutation
* Law: mistakes made by a trial court
* Statistics: an error (or residual) is not a "mistake"...
    * but rather a difference between a computed, estimated, or measured value and the accepted true, specified, or theoretically correct value

---

# What _isn't_ an error? A successfully terminated process!

<iframe src="https://doc.rust-lang.org/std/process/struct.ExitCode.html#examples" class="h-full"></iframe>

---

# What _isn't_ an error? A panic!

```rust
panic!("This is not an error!");
```

---

## ...unless you catch it
<iframe src="https://doc.rust-lang.org/std/panic/fn.take_hook.html#examples" class="h-full"></iframe>

---

## `Result::Err` and `std::error::Error`


---

`Result::Err` is quite similar to `Option`:

```rust
//       keyword
//       |      Type of returned value that will be
//       |      | held by `Some(T)`
//       v      v
pub enum Option<T> { // < Body of enum option,
    None,            // | When Option is unwrapped:
    Some(T),         // | it will be either `None`
}                    // < or `Some(T)`
```

---

the `T` is now accompanied by an `E`:

```rust
//       keyword
//       |      `Ok(T)`
//       |      | `Err(E)`
//       v      v  v
pub enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

* in essence, an _error_ in rust (usually) is something you eventuall expect to return in an `Result::Err`
* the three following examples will be dealing with the inner `E` of a `Result<T, E>`

---

## `dyn Error`

the `std::error::Error` Trait

```rust
pub trait Error: Debug + Display {
    // Provided methods
    fn source(&self) -> Option<&(dyn Error + 'static)> { ... }
    // these two are deprecated
    fn description(&self) -> &str { ... }
    fn cause(&self) -> Option<&dyn Error> { ... }
    // this one is nightly only
    fn provide<'a>(&'a self, request: &mut Request<'a>) { ... }
}
```

---

# `dyn Error`

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    Err("throw this str into a box!".into())
}
```

---

<iframe src="https://docs.rs/anyhow/latest/anyhow/struct.Error.html" class="h-full"></iframe>

---

## type erasure

```rust
// anyhow::Error
pub struct Error {
    inner: Own<ErrorImpl>,
}

impl<E> ErrorImpl<E> {
    fn erase(&self) -> Ref<ErrorImpl> {
        // Erase the concrete type of E but preserve the vtable in self.vtable
        // for manipulating the resulting thin pointer. This is analogous to an
        // unsize coercion.
        Ref::new(self).cast::<ErrorImpl>()
    }
}
```

---

## concrete types

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DataStoreError {
    #[error("data store disconnected")]
    Disconnect(#[from] io::Error),
    #[error("the data for key `{0}` is not available")]
    Redaction(String),
    #[error("invalid header (expected {expected:?}, found {found:?})")]
    InvalidHeader {
        expected: String,
        found: String,
    },
    #[error("unknown data store error")]
    Unknown,
}

fn main() -> Result<(), DataStoreError> {
    Err(DataStoreError::Unknown)
}
```

---

To review:

* dynamic dispatch/opaque type:
  `dyn Error` (`std::error::Error`)
* `enum Error` concrete type (oftentimes an `enum`)
* `Erase::err(self).cast::<OwnedDataStructure>()` (type erasure/corecion)


---

## The `?` operator

https://doc.rust-lang.org/std/ops/trait.Try.html

---

<iframe src="https://google.github.io/comprehensive-rust/error-handling/try.html" class="h-full"></iframe>

---

## With `Option`

```rust
fn try_1() -> Option<usize> {
    let _ = Some(1)?;
    None
}

fn try_2() -> Option<usize> {
    let _ = None?;
    Some(1)
}
```

---

## `error-stack`

https://hash.dev/blog/announcing-error-stack

---

`error_stack::Report`:

```rust
pub struct Report<C> {
    // The vector is boxed as this implies a memory footprint equal to a single pointer size
    // instead of three pointer sizes. Even for small `Result::Ok` variants, the `Result` would
    // still have at least the size of `Report`, even at the happy path. It's unexpected, that
    // creating or traversing a report will happen in the hot path, so a double indirection is
    // a good trade-off.
    #[allow(clippy::box_collection)]
    pub(super) frames: Box<Vec<Frame>>,
    _context: PhantomData<fn() -> *const C>,
}
```

---

`error_stack::Context`:

```rust
pub trait Context: Display + Debug + Send + Sync + 'static {
    // Provided method
    fn provide<'a>(&'a self, request: &mut Request<'a>) { ... }
}
```

---

```rust
impl<C> Report<C> {
    pub fn change_context<T>(mut self, context: T) -> Report<T>
    where
        T: Context,
    {
        let old_frames = mem::replace(self.frames.as_mut(), Vec::with_capacity(1));
        let context_frame = vec![Frame::from_context(context, old_frames.into_boxed_slice())];
        self.frames.push(Frame::from_attachment(
            *Location::caller(),
            context_frame.into_boxed_slice(),
        ));
        Report {
            frames: self.frames,
            _context: PhantomData,
        }
    }
}
```

* `Frame`s?
* `Location`?

---

<iframe src="https://docs.rs/error-stack/latest/error_stack/iter/struct.Frames.html#example" class="h-full"></iframe>

---

<iframe src="https://doc.rust-lang.org/std/panic/struct.Location.html" class="h-full"></iframe>

---

`Location` and `#[track_caller]`:

<iframe src="https://doc.rust-lang.org/reference/attributes/codegen.html#the-track_caller-attribute" class="h-full"></iframe>

---

```rust
impl<T, C> ResultExt for core::result::Result<T, C>
where
    C: Context,
{
    type Context = C;
    type Ok = T;

    #[track_caller]
    fn attach<A>(self, attachment: A) -> Result<T, C>
    where
        A: Send + Sync + 'static,
    {
        match self {
            Ok(value) => Ok(value),
            Err(error) => Err(Report::new(error).attach(attachment)),
        }
    }
}
```

---

## `error-stack` and intent

<iframe src="https://docs.rs/error-stack/latest/error_stack/index.html#initializing-a-report" class="h-full"></iframe>

---

## `bigerror` motications

- `thiserror` enum trees
- `error-stack` verbosity
- trait speicialization
- nightly features


---

```rust
pub trait OptionReport<T>
where
    Self: Sized,
{
    fn expect_or(self) -> Result<T, Report<NotFound>>;
    fn expect_kv<K, V>(self, key: K, value: V) -> Result<T, Report<NotFound>>
    where
        K: Display,
        V: Display;
    fn expect_field(self, field: &'static str) -> Result<T, Report<NotFound>>;

    #[inline]
    #[track_caller]
    fn expect_kv_dbg<K, V>(self, key: K, value: V) -> Result<T, Report<NotFound>>
    where
        K: Display,
        V: Debug,
    {
        self.expect_kv(key, Dbg(value))
    }

    #[inline]
    #[track_caller]
    fn expect_by<K: Display>(self, key: K) -> Result<T, Report<NotFound>> {
        self.expect_kv(Index(key), ty!(T))
    }
}
```

---

```rust
pub trait EnsureCount<T>
where
    Self: Sized,
{
    fn len(&self) -> usize;
    fn is_empty(&self) -> bool {
        self.len() == 0
    }
    // first (and hopefully only) unordered element
    fn into_only(self) -> Option<T>;
    // ensures a collection has only one element
    fn only_one(self) -> Result<T, Report<AssertionError>> {
        ensure!(self.len() == 1, self.err("expected exactly one element"));
        Ok(self.into_only().unwrap())
    }
    // ensures a collection has one or zero elements
    fn one_or_none(self) -> Result<Option<T>, Report<AssertionError>> {
        ensure!(
            self.len() == 1 || self.is_empty(),
            self.err("expected one or zero elements")
        );
        Ok(self.into_only())
    }
    fn one_or_more(&self) -> Result<(), Report<AssertionError>> {
        ensure!(!self.is_empty(), self.err("expected one or more elements"));
        Ok(())
    }
    fn err(&self, msg: &'static str) -> Report<AssertionError> {
        AssertionError::with_kv(ty!(Self), msg).attach_kv("len", self.len())
    }
}
```

---

https://github.com/dtolnay/case-studies/tree/master/autoref-specialization

---

## `bigerror` future considerations

- `_lazy` variant coverage
- errors that are not `Send + Sync`
- colour formatting
