---
layout: post
title:  "Rust SmallVec"
subtitle: "An array-backed alternative to the standard library vector class, expandable with heap allocations."
date:   2023-07-12 14:30:00
categories: [seahorn, rust]
---


`SmallVec` is an alternative to `tinyvec` that provides an array-backed vector implementation but also allows the vector to be expanded at run time by allocating more memory on the heap. This saves time over a normal vector by minimizing the number of allocations that must be done and keeping most of the vector in a fixed capacity array.

To test whether `SmallVec` could be used for verification of Rust programs using `SeaHorn`, I created verification jobs for each of the methods.

## Limitations

Since the fixed capacity of `SmallVec` is constant and must be known at compile time, it isn't possible to verify the class for non-deterministic capacities. In addition, since `SeaHorn` has issues with some of the tests, I had to limit the capacity of the vector to 1 in some cases. This means that the tests are not as general as they could be.

## Test Format

I wrote the tests for `SmallVec` in a very similar format to the tests for `tinyvec`. I identified all different ways they could behave under different circumstances. Some examples include empty vectors and vectors at capacity that need dynamic allocation for further operations. I also identified where methods should panic and tested for this by putting them at the end of a test with a `sea::sassert!(false)` directly after. The only way this assertion coud be reached is if the operations executed successfuly, allowing it to test whether they panicked or not.

Example: `test_pop`

```rust
fn test_pop() {
    let mut v: SmallVec<[u32; 8]> = SmallVec::new();

    let len: usize = sea::nd_usize();
    sea::assume(len > 0 && len <= 8);

    for _i in 0..len {
        v.push(sea::nd_u32());
    }

    for i in 0..len {
        v.pop();
        sea::sassert!(v.len() == len - i - 1);
    }

    let result: Option<u32> = v.pop();
    sea::sassert!(result.is_none());
}
```

In this example, I push non-deterministic values to a vector in a loop of non-deterministic size. I then pop each of the values off the vector and assert that the length is correct after each pop. Finally, I pop one more value off the vector when it is empty and assert that the result is `None`.

## Issues

`SeaHorn` has issues with verifying `SmallVec` whenever it goes beyond its fixed capacity and requires more memory allocated on the heap. Even if only one additional space in the vector is required, a very large call to `alloca` is generated, causing `SeaHorn` to take excessive time to get through this stage and in some cases failing. I have attempted to reduce the size of the vectors and to make the verification jobs more simple, however I haven't been able able to come up with a solution yet.

Another issue that comes up with certain tests such as `test_push` is that some assertions are not reached if the capacity of the vector is too large.

```rust
fn test_push() {
    const CAP: usize = 1;
    let mut v: SmallVec<[u32; CAP]> = SmallVec::new();

    let len: usize = sea::nd_usize();
    sea::assume(len <= CAP);

    for i in 0..len {
        v.push(sea::nd_u32());
        sea::sassert!(v.len() == i + 1);
    }

    sea::sassert!(v.len() == len);
    sea::sassert!(v.capacity() == CAP);

    sea::sea_printf!("len", len);
    if len == CAP {
        v.push(sea::nd_u32());
        // Only reached when CAP == 1
        sea::sassert!(v.len() == CAP + 1);
        sea::sassert!(v.capacity() > CAP);
    }
}
```

In this code, the final two assertions inside the if statement are only reached when the vector has a capacity of 1. I am still looking into this issue to determine what might be causing it.

## Source Code

Full source code for the tests can be found [here](https://github.com/thomashart17/c-rust/tree/smallvec/src/rust-jobs/smallvec).
