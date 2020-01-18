---
title: "Reviewing Unsafe Rust"
date: "2020-01-18T20:38:06+08:00"
draft: true
---

There has recently been a bit of a kerfuffle in the Rust community around the
actix-web project. Rather than talking about the public outcry and nasty
things being said on Reddit or the author's fast-and-loose attitude towards
writing `unsafe` code (Steve Klabnik has [already explained it][sad-day] much
better than I could) I would like to discuss some technical aspects of `unsafe`
Rust.

In particular a lot of people say we should be reviewing our dependencies for
possibly unsound code, but nobody seems to explain *how* such a review is done
or how to reason about correctness.

There's also a tendency to understate how much effort is required to review code
in enough detail that the review can be relied on. To that end, the [crev][crev]
project has done a lot of work to help distribute the review effort and build a
*Web of Trust* system.

I'll also be keeping track of the time taken using an app called
[clockify][clockify], that way at the end we can see a rough breakdown of
time spent and get a more realistic understanding of the effort required to
review code.

{{% notice note %}}
The code written in this article is available [on GitHub][repo]. Feel free to
browse through and steal code or inspiration.

If you found this useful or spotted a bug, let me know on the blog's
[issue tracker][issue]!

[repo]: https://github.com/Michael-F-Bryan/💩🔥🦀
[issue]: https://github.com/Michael-F-Bryan/adventures.michaelfbryan.com
{{% /notice %}}

## Introducing the `anyhow` Crate

Over the past couple months the [`anyhow`][anyhow] and [`thiserror`][thiserror]
crates from [@dtolnay][dtolnay] have helped simplify error handling in Rust a
lot.

The `thiserror` crate is a procedural macro for automating the implementation of
`std::error::Error` and isn't overly interesting for our purposes.

On the other hand, the `anyhow` crate presents a custom `Error` type and uses a
`unsafe` code under the hood to implement nice things like downcasting and
representing an `anyhow::Error` with a thin pointer.

Before we start the review proper, it's worth looking at the top-level docs and
examples so we can get a rough understanding of how things fit together.

Everything in this article will be done using version 1.0.26 of `anyhow`.

First we'll use `cargo-crev` to drop into a directory with the crate's source
code.

```console
$ cargo crev crate goto anyhow 1.0.26
Opening shell in: /home/michael/.cargo/registry/src/github.com-1ecc6299db9ec823/anyhow-1.0.26
Use `exit` or Ctrl-D to return to the original project.
Use `review` and `flag` without any arguments to review this crate.
$ code .
```

Next we can open the API docs in a browser and have a look around. This crate
is really well documented, so that makes things a lot easier!

```console
$ cargo doc --open
   Compiling anyhow v1.0.26 (/home/michael/.cargo/registry/src/github.com-1ecc6299db9ec823/anyhow-1.0.26)
 Documenting anyhow v1.0.26 (/home/michael/.cargo/registry/src/github.com-1ecc6299db9ec823/anyhow-1.0.26)
    Finished dev [unoptimized + debuginfo] target(s) in 4.03s
     Opening /home/michael/.cargo/registry/src/github.com-1ecc6299db9ec823/anyhow-1.0.26/target/doc/anyhow/index.html
```

{{% notice tip %}}
For those of you following along at home you can also [view the docs
online][docs].

Although if you're doing a proper review I would strongly recommend to build
everything from source instead of relying on 3rd parties. I doubt docs.rs is
trying to deceive us, but it's always good to know that everything you are
looking at comes from the same set of code.

[docs]: https://docs.rs/anyhow/1.0.26/anyhow/
{{% /notice %}}

The crate only exports two types, an `Error` and a helper type for iterating
over a chain of errors.

However looking at the files in the project shows a lot more than just two
files...

```console
$
tree -I 'target|tests'
.
├── build.rs
├── Cargo.lock
├── Cargo.toml
├── Cargo.toml.orig
├── LICENSE-APACHE
├── LICENSE-MIT
├── README.md
└── src
    ├── backtrace.rs
    ├── chain.rs
    ├── context.rs
    ├── error.rs
    ├── fmt.rs
    ├── kind.rs
    ├── lib.rs
    ├── macros.rs
    └── wrapper.rs

1 directory, 16 files
$ tokei --exclude target --exclude tests
-------------------------------------------------------------------------------
 Language            Files        Lines         Code     Comments       Blanks
-------------------------------------------------------------------------------
 Markdown                1          163          163            0            0
 Rust                   10         2257         1140          932          185
 TOML                    1           37           21           11            5
-------------------------------------------------------------------------------
 Total                  12         2457         1324          943          190
-------------------------------------------------------------------------------
```

We can also do a quick search for `"unsafe"` to see how much `unsafe` code there
is and how it's being used.

```console
$ rg unsafe --glob '!target'
src/error.rs
90:        unsafe { Error::construct(error, vtable, backtrace) }
111:        unsafe { Error::construct(error, vtable, backtrace) }
132:        unsafe { Error::construct(error, vtable, backtrace) }
154:        unsafe { Error::construct(error, vtable, backtrace) }
176:        unsafe { Error::construct(error, vtable, backtrace) }
184:    unsafe fn construct<E>(
285:        unsafe { Error::construct(error, vtable, backtrace) }
366:        unsafe {
434:        unsafe {
448:        unsafe {
498:        unsafe {
510:    object_drop: unsafe fn(Box<ErrorImpl<()>>),
511:    object_ref: unsafe fn(&ErrorImpl<()>) -> &(dyn StdError + Send + Sync + 'static),
513:    object_mut: unsafe fn(&mut ErrorImpl<()>) -> &mut (dyn StdError + Send + Sync + 'static),
514:    object_boxed: unsafe fn(Box<ErrorImpl<()>>) -> Box<dyn StdError + Send + Sync + 'static>,
515:    object_downcast: unsafe fn(&ErrorImpl<()>, TypeId) -> Option<NonNull<()>>,
516:    object_drop_rest: unsafe fn(Box<ErrorImpl<()>>, TypeId),
520:unsafe fn object_drop<E>(e: Box<ErrorImpl<()>>) {
528:unsafe fn object_drop_front<E>(e: Box<ErrorImpl<()>>, target: TypeId) {
538:unsafe fn object_ref<E>(e: &ErrorImpl<()>) -> &(dyn StdError + Send + Sync + 'static)
548:unsafe fn object_mut<E>(e: &mut ErrorImpl<()>) -> &mut (dyn StdError + Send + Sync + 'static)
557:unsafe fn object_boxed<E>(e: Box<ErrorImpl<()>>) -> Box<dyn StdError + Send + Sync + 'static>
566:unsafe fn object_downcast<E>(e: &ErrorImpl<()>, target: TypeId) -> Option<NonNull<()>>
583:unsafe fn context_downcast<C, E>(e: &ErrorImpl<()>, target: TypeId) -> Option<NonNull<()>>
603:unsafe fn context_drop_rest<C, E>(e: Box<ErrorImpl<()>>, target: TypeId)
626:unsafe fn context_chain_downcast<C>(e: &ErrorImpl<()>, target: TypeId) -> Option<NonNull<()>>
643:unsafe fn context_chain_drop_rest<C>(e: Box<ErrorImpl<()>>, target: TypeId)
693:        unsafe { &*(self as *const ErrorImpl<E> as *const ErrorImpl<()>) }
701:        unsafe { &*(self.vtable.object_ref)(self) }
708:        unsafe { &mut *(self.vtable.object_mut)(self) }
762:        unsafe {
```

It looks like all the `unsafe` code is in `src/error.rs` and the frequent
references to a `vtable` indicate this is probably related to some sort of
hand-coded trait object.

## Beginning the Review

We'll start off the review by checking out `lib.rs`.

After scrolling past almost 200 lines of top-level docs we come across our first
interesting block of code.

```rust
// src/lib.rs

mod alloc {
    #[cfg(not(feature = "std"))]
    extern crate alloc;

    #[cfg(not(feature = "std"))]
    pub use alloc::boxed::Box;

    #[cfg(feature = "std")]
    pub use std::boxed::Box;
}
```

Now the code itself is quite boring, but the fact that it exists says a lot
about this crate. Namely that they're going to great lengths to ensure `anyhow`
is usable without the standard library.

This `alloc` module allows code to use `crate::alloc::Box` when boxing things
instead of relying on the `Box` added to the prelude by `std` (which isn't in
the prelude for `#[no_std]` crates).

You can also see they're providing a polyfill for `std::error::Error` when
compiled without the standard library.

```rust
// src/lib.rs

#[cfg(feature = "std")]
use std::error::Error as StdError;

#[cfg(not(feature = "std"))]
trait StdError: Debug + Display {
    fn source(&self) -> Option<&(dyn StdError + 'static)> {
        None
    }
}
```

Next comes the declaration for `Error` itself.

```rust
// lib.rs

pub struct Error {
    inner: ManuallyDrop<Box<ErrorImpl<()>>>,
}
```

One of the distinctions raised in `Error`'s documentation is that it's
represented using a narrow pointer. It looks like `ErrorImpl<()>` is a concrete
struct which holds all the implementation details for `Error`.

The `ManuallyDrop` is certainly interesting. Straight away this tells me that
`Error` will be explicitly implementing `Drop` and probably doing some `unsafe`
shenanigans to make sure our `ErrorImpl` is destroyed properly.

I'll jump down to around line 540 where things start getting interesting again.

```rust
// lib.rs

pub trait Context<T, E>: context::private::Sealed {
    /// Wrap the error value with additional context.
    fn context<C>(self, context: C) -> Result<T, Error>
    where
        C: Display + Send + Sync + 'static;

    /// Wrap the error value with additional context that is evaluated lazily
    /// only once an error does occur.
    fn with_context<C, F>(self, f: F) -> Result<T, Error>
    where
        C: Display + Send + Sync + 'static,
        F: FnOnce() -> C;
}
```

This is the declaration for the `Context` trait. It's a helper for converting
something like `Result<T, E>` or `Option<T>` into a `Result<T, Error>`.

Notably, the trait uses [the Sealed Pattern][sealed] so it's impossible for
downstream users to implement `Context` on their own types. Presumably this is
to allow `Context` to be extended in the future without breaking backwards
compatibility.

Sealing a trait can also be used to ensure it is only implemented for a specific
set of types or to ensure a particular invariant is upheld, but that's probably
not the case here.

Finally, we get to a peculiar `private` module at the bottom of the file.

```rust
// src/lib.rs

// Not public API. Referenced by macro-generated code.
#[doc(hidden)]
pub mod private {
    use crate::Error;
    use core::fmt::{Debug, Display};

    #[cfg(backtrace)]
    use std::backtrace::Backtrace;

    pub use core::result::Result::Err;

    #[doc(hidden)]
    pub mod kind {
        pub use crate::kind::{AdhocKind, TraitKind};

        #[cfg(feature = "std")]
        pub use crate::kind::BoxedKind;
    }

    pub fn new_adhoc<M>(message: M) -> Error
    where
        M: Display + Debug + Send + Sync + 'static,
    {
        Error::from_adhoc(message, backtrace!())
    }
}
```

It looks like code generated by macros will use the `private` module when
constructing errors (i.e. by calling `$crate::private::new_adhoc()`).

## The Real Meat and Potatoes

Now we've checked out `lib.rs`, the use of `ManuallyDrop` has made me curious
to see how `Error` is implemented under the hood.

It looks like `error.rs` contains the majority of this crate's functionality.
Weighing in at a whopping 794 lines with 30 uses of the `unsafe` keyword, this
may take a while...

## Time Taken

## Conclusions

[sad-day]: https://words.steveklabnik.com/a-sad-day-for-rust
[crev]: https://github.com/crev-dev/crev
[clockify]: https://clockify.me/
[anyhow]: https://github.com/dtolnay/anyhow
[thiserror]: https://github.com/dtolnay/thiserror
[dtolnay]: https://github.com/dtolnay
[sealed]: https://rust-lang.github.io/api-guidelines/future-proofing.html#sealed-traits-protect-against-downstream-implementations-c-sealed