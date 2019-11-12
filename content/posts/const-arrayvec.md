---
title: "Implementing ArrayVec Using Const Generics"
date: "2019-11-12T20:57:53+08:00"
draft: true
tags:
- rust
---

If you've ever done much embedded programming in Rust, you've most probably run
across the [`arrayvec`][arrayvec] crate before. It's awesome. The main purpose
of the crate is to provide the `ArrayVec` type, which is essentially like
`Vec<T>` from the standard library, but backed by an array instead of some
memory on the heap.

One of the problems I ran into while writing the *Motion Planning* chapter of my
[Adventures in Motion Control][aimc] was deciding how far ahead my motion
planner should plan.

The *Adventures in Motion Control* series is targeting a platform without an
allocator, so the number of moves will be determined at compile time. I *could*
pluck a number out of thin air and say *"she'll be right"*, but there's also
this neat feature on *nightly* at the moment called [*"Const Generics"*][cg]...

{{< figure 
    src="https://imgs.xkcd.com/comics/nerd_sniping.png" 
    link="https://xkcd.com/356/" 
    caption="(obligatory XKCD reference)" 
    alt="Nerd Sniping" 
>}}

## Getting Started

Okay, so the first thing we'll need to do is create a crate and enable this 
`const_generics` feature.

When creating a new project I like to use [`cargo generate`][cargo-generate]
and a template repository. It just saves needing to manually copy license info
from another project.

```console
$ cargo generate --git https://github.com/Michael-F-Bryan/github-template --name const-arrayvec
 Creating project called `const-arrayvec`...
 Done! New project created /home/michael/Documents/const-arrayvec
```

And we'll need to update `lib.rs`:

```rust
// src/lib.rs

//! An implementation of the [arrayvec](https://crates.io/crates/arrayvec) crate
//! using *Const Generics*.

#![no_std]
#![feature(const_generics)]
```

It's around this time that I'll start a second terminal and use [`cargo watch`][cw]
to run `cargo build`, `cargo test`, and `cargo doc` in the background.

```console
$ cargo watch --clear 
    -x "check --all" 
    -x "test --all" 
    -x "doc --document-private-items --all" 
    -x "build --release --all"
warning: the feature `const_generics` is incomplete and may cause the compiler to crash
 --> src/lib.rs:2:12
  |
2 | #![feature(const_generics)]
  |            ^^^^^^^^^^^^^^
  |
  = note: `#[warn(incomplete_features)]` on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.02s
```

Well that's encouraging. You know a feature *must* be unstable when even 
*nightly* warns you it may crash the compiler...

{{% notice tip %}}
You can "fix" this compiler warning by adding a `#![allow(incomplete_features)]`
to the top of `lib.rs`.
{{% /notice %}}

Now we've got a crate, we can start implementing our `ArrayVec` type.

The first decision we need to make is how values will be stored within our
`ArrayVec` the naive implementation would use a simple `[T; N]` array, but that
only works when `T` has a nice default value and no destructor. Instead, what
we're really doing is storing an array of possibly initialized `T`'s... Which
is exactly what [`core::mem::MaybeUninit`][MaybeUninit] is designed for.

```rust
// src/lib.rs

use core::mem::MaybeUninit;

pub struct ArrayVec<T, const N: usize> {
    items: [MaybeUninit<T>; N],
    length: usize,
}
```

Next we'll need to give `ArrayVec` a constructor. This is what I *would* like
to write...

```rust
// src/lib.rs

impl<T, const N: usize> ArrayVec<T, { N }> {
    pub const fn new() -> ArrayVec<T, { N }> {
        ArrayVec {
            items: [MaybeUninit::uninit(); N],
            length: 0,
        }
    }
}
```

... But unfortunately it doesn't seem to be implemented yet (see 
[this u.rl.o post][forum]).

```
    Checking const-arrayvec v0.1.0 (/home/michael/Documents/const-arrayvec)
error: array lengths can't depend on generic parameters
  --> src/lib.rs:15:44
   |
15 |             items: [MaybeUninit::uninit(); N],
   |                                            ^

error: aborting due to previous error

error: could not compile `const-arrayvec`.
```

Instead we'll need to drop the `const fn` for now and find a different way of
creating our array of uninitialized data.

```rust
// src/lib.rs

impl<T, const N: usize> ArrayVec<T, { N }> {
    pub fn new() -> ArrayVec<T, { N }> {
        unsafe {
            ArrayVec {
                // this is safe because we've asked for a big block of
                // uninitialized memory which will be treated as
                // an array of uninitialized items,
                // which perfectly valid for [MaybeUninit<_>; N]
                items: MaybeUninit::uninit().assume_init(),
                length: 0,
            }
        }
    }
}
```

While we're at it, because we're implementing a collection we should add `len()`
and friends.

```rust
// src/lib.rs

impl<T, const N: usize> ArrayVec<T, { N }> {
    ...

    pub const fn len(&self) -> usize { self.length }

    pub const fn is_empty(&self) -> bool { self.len() == 0 }

    pub const fn capacity(&self) -> usize { N }

    pub const fn is_full(&self) -> bool { self.len() == self.capacity() }
}
```

We also want a way to get a raw pointer to the first element in the underlying
buffer. This will be important when we actually need to read data.

```rust
// src/lib.rs

impl<T, const N: usize> ArrayVec<T, { N }> {
    ...

    pub fn as_ptr(&self) -> *const T { self.items.as_ptr() as *const T }

    pub fn as_mut_ptr(&mut self) -> *mut T { self.items.as_mut_ptr() as *mut T }
}
```

## The Basic Operations

About the most basic operation for a `Vec`-like container to support is adding
and removing items, so that's what we'll be implementing next.

As you may have guessed, this crate will do a lot of work with possibly
initialized memory so there'll be a decent chunk of `unsafe` code. 

```rust
// src/lib.rs

impl<T, const N: usize> ArrayVec<T, { N }> {
    ...

    /// Add an item to the end of the array without checking the capacity.
    /// 
    /// # Safety
    /// 
    /// It is up to the caller to ensure the vector's capacity is suitably 
    /// large.
    /// 
    /// This method uses *debug assertions* to detect overflows in debug builds.
    pub unsafe fn push_unchecked(&mut self, item: T) {
        debug_assert!(!self.is_full());
        let len = self.len();

        // index into the underlying array using pointer arithmetic and write
        // the item to the correct spot
        self.as_mut_ptr().add(len).write(item);

        // only now can we update the length
        self.set_len(len + 1);
    }

    /// Set the vector's length without dropping or moving out elements.
    /// 
    /// # Safety
    /// 
    /// This method is `unsafe` because it changes the number of "valid"
    /// elements the vector thinks it contains, without adding or removing any
    /// elements. Use with care.
    pub unsafe fn set_len(&mut self, new_length: usize) {
        debug_assert!(new_length < self.capacity());
        self.length = new_length;
    }
}
```

The `push_unchecked()` and `set_len()` methods should be fairly descriptive,
so I'll just let you read the code. Something to note 

{{% notice note %}}
You would have noticed that the `unsafe` functions have a `# Safety` section in
their doc-comments specifying various assumptions and invariants that must be
upheld. 

This is quite common when writing `unsafe` code, and is actually
[part of the Rust API guidelines][guidelines]. I would recommend giving that
document a quick read if you haven't already.

[guidelines]: https://rust-lang.github.io/api-guidelines/documentation.html#c-failure
{{% /notice %}}

We also need to expose a safe `push()` method. Preferably one which will return
the original item when there is no more space.

```rust
```rust
// src/lib.rs

use core::fmt::{self, Display, Formatter};

impl<T, const N: usize> ArrayVec<T, { N }> {
    ...

    /// Add an item to the end of the vector.
    ///
    /// # Examples
    ///
    /// ```rust
    /// # use const_arrayvec::ArrayVec;
    /// let mut vector: ArrayVec<u32, 5> = ArrayVec::new();
    ///
    /// assert!(vector.is_empty());
    ///
    /// vector.push(42);
    ///
    /// assert_eq!(vector.len(), 1);
    /// assert_eq!(vector[0], 42);
    /// ```
    pub fn push(&mut self, item: T) -> Result<(), CapacityError<T>> {
        if self.is_full() {
            Err(CapacityError(item))
        } else {
            unsafe {
                self.push_unchecked(item);
                Ok(())
            }
        }
    }
}

#[derive(Debug, Copy, Clone, PartialEq, Eq, Hash)]
pub struct CapacityError<T>(pub T);

impl<T> Display for CapacityError<T> {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        write!(f, "Insufficient capacity")
    }
}
```

While we're at it, we should add a `pop()` method. This one is quite similar,
except implemented in reverse (i.e. the length is decremented and we read from
the array).

```rust
// src/lib.rs

use core::ptr;

impl<T, const N: usize> ArrayVec<T, { N }> {
    ...

    /// Remove an item from the end of the vector.
    ///
    /// # Examples
    ///
    /// ```rust
    /// # use const_arrayvec::ArrayVec;
    /// let mut vector: ArrayVec<u32, 5> = ArrayVec::new();
    ///
    /// vector.push(12);
    /// vector.push(34);
    ///
    /// assert_eq!(vector.len(), 2);
    ///
    /// let got = vector.pop();
    ///
    /// assert_eq!(got, Some(34));
    /// assert_eq!(vector.len(), 1);
    /// ```
    pub fn pop(&mut self) -> Option<T> {
        if self.is_empty() {
            return None;
        }

        unsafe {
            let new_length = self.len() - 1;
            self.set_len(new_length);
            Some(ptr::read(self.as_ptr().add(new_length)))
        }
    }
}
```

Some more relatively straightforward methods are `clear()` and `truncate()` for 
shortening the vector and dropping any items after the new end.

```rust
// src/lib.rs

impl<T, const N: usize> ArrayVec<T, { N }> {
    ...

    /// Shorten the vector, keeping the first `new_length` elements and dropping
    /// the rest.
    pub fn truncate(&mut self, new_length: usize) {
        unsafe {
            if new_length < self.len() {
                let start = self.as_mut_ptr().add(new_length);
                let num_elements_to_remove = self.len() - new_length;
                let tail: *mut [T] =
                    slice::from_raw_parts_mut(start, num_elements_to_remove);

                self.set_len(new_length);
                ptr::drop_in_place(tail);
            }
        }
    }

    /// Remove all items from the vector.
    pub fn clear(&mut self) { self.truncate(0); }
}
```

{{% notice note %}}
Note the use of `core::ptr::drop_in_place()`, this will call the destructor of
every item in the `tail` and leave them in a logically uninitialized state.
{% /notice %}

Next comes one of the trickier methods for our collection, `try_insert()`. When
inserting, after doing a couple bounds checks we'll need to move everything
after the insertion point over one space. Because the memory we're copying 
*from* overlaps with the memory we're copying *to*, we need to use the less
performant `core::ptr::copy()` (the Rust version of C's `memmove()`) instead of
`core::ptr::copy_non_overlapping()` (equivalent of C's `memcpy()`).

Most of this code is lifted straight from [`alloc::vec::Vec::insert()`][vec].

```rust
// src/lib.rs

macro_rules! out_of_bounds {
    ($method:expr, $index:expr, $len:expr) => {
        panic!(
            concat!(
                "ArrayVec::",
                $method,
                "(): index {} is out of bounds in vector of length {}"
            ),
            $index, $len
        );
    };
}

impl<T, const N: usize> ArrayVec<T, { N }> {
    ...

    pub fn try_insert(
        &mut self,
        index: usize,
        item: T,
    ) -> Result<(), CapacityError<T>> {
        let len = self.len();

        // bounds checks
        if index > self.len() {
            out_of_bounds!("try_insert", index, len);
        }
        if self.is_full() {
            return Err(CapacityError(item));
        }

        unsafe {
            // The spot to put the new value
            let p = self.as_mut_ptr().add(index);
            // Shift everything over to make space. (Duplicating the
            // `index`th element into two consecutive places.)
            ptr::copy(p, p.offset(1), len - index);
            // Write it in, overwriting the first copy of the `index`th
            // element.
            ptr::write(p, item);
            // update the length
            self.set_len(len + 1);
        }

        Ok(())
    }
}
```

## Implementing Useful Conversion Traits

We're now at the point where we can implement `Deref` and `DerefMut` to make our
`ArrayVec` more array-like.

The implementation itself is quite boring (we just call
`core::slice::from_raw_parts()`), but this is the first step on the way to being
a first-class container.

```rust
// src/lib.rs

use core::ops::{Deref, DerefMut};

impl<T, const N: usize> Deref for ArrayVec<T, { N }> {
    type Target = [T];

    fn deref(&self) -> &Self::Target {
        unsafe { slice::from_raw_parts(self.as_ptr(), self.len()) }
    }
}

impl<T, const N: usize> DerefMut for ArrayVec<T, { N }> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        unsafe { slice::from_raw_parts_mut(self.as_mut_ptr(), self.len()) }
    }
}
```

[arrayvec]: https://crates.io/crates/arrayvec
[aimc]: http://adventures.michaelfbryan.com/tags/aimc
[cg]: https://github.com/rust-lang/rust/issues/44580
[cargo-generate]: https://crates.io/crates/cargo-generate
[cw]: https://crates.io/crates/cargo-watch
[MaybeUninit]: https://doc.rust-lang.org/core/mem/union.MaybeUninit.html
[forum]: https://doc.rust-lang.org/core/mem/union.MaybeUninit.html
[vec]: https://github.com/rust-lang/rust/blob/a19f93410d4315408f8775e1be29536302adc223/src/liballoc/vec.rs#L993-L1016