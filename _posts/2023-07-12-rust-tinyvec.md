---
layout: post
title:  "Rust TinyVec"
subtitle: "An array-backed alternative to the standard library vector class."
date:   2023-07-12 14:00:00
categories: [seahorn, rust]
---

To provide a more efficient alternative to the `Vec` class from `std`, the `tinyvec` crate provides the `ArrayVec` class. `ArrayVec` is an array-backed vector implementation that saves time by removing the need for memory allocations when there is a hard cap on vector capacity.

To improve verification times for Rust programs, I looked at verifying the `ArrayVec` class as a potential replacement for `Vec`.

## Limitations

Since the capacity of `ArrayVec` is based on the backing array, it must be known at compile time and thus it isn't possible to verify the class for non-deterministic sizes. For my verification jobs, I used arrays with a capacity of 8 and created non-deterministic length values less than or equal to 8, to allow different sized vectors to be tested.

Certain jobs were taking excessive amounts of time to run, so I had to use the `horn-explicit-sp0` option to run the verificiation, making it less general.

## Test Format

To verify each of the methods in `ArrayVec`, I identified all different ways they could behave under different circumstances. Some examples include empty vectors and vectors at capacity. I also identified where methods should panic and tested for this by putting them at the end of a test with a `sea::sassert!(false)` directly after. The only way this assertion coud be reached is if the operations executed successfuly, allowing it to test whether they panicked or not.

Example: `test_push`

```rust
fn test_push() {
    let mut v: ArrayVec<[u32; 8]> = ArrayVec::new();
    let len: usize = sea::nd_usize();
    sea::assume(len <= 8);

    for i in 0..len {
        v.push(sea::nd_u32());
        sea::sassert!(v.len() == i + 1);
    }

    sea::sassert!(v.len() == len);
    sea::sassert!(v.capacity() == 8);

    if len == 8 {
        // Vector is at capacity, so push should panic.
        v.push(sea::nd_u32());

        // This assertion should not be reachable since the previous push panics.
        sea::sassert!(false);
    }
}
```

In this example, I push non-deterministic values to a vector and have assertions to verify that it has the correct length and capacity after these pushes. If the length of the vector is equal to the capacity, I push another value and assert false to ensure the operation panics.

## Issues

When working on verifying the `splice` method, I came across an interesting issue. Using an inclusive range:

```rust
0..=2
```

rather than an exclusive range:

```rust
0..3
```

for the replacement parameter caused the verification to take excessive amounts of time. This was fixed by using the `horn-explicit-sp0` option.

## Source Code

Full source code for the tests can be found [here](https://github.com/thomashart17/c-rust/tree/main/src/rust-jobs/tinyvec-arrayvec).
