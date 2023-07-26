---
layout: post
title:  "Rust SmallVec"
subtitle: "An array-backed alternative to the standard library vector class, expandable with heap allocations."
date:   2023-07-26 18:00:00
categories: [seahorn, rust]
---


`SmallVec` is an alternative to `tinyvec` that provides an array-backed vector implementation but also allows the vector to be expanded at run time by allocating more memory on the heap. This saves time over a normal vector by minimizing the number of allocations required and keeping most of the vector in a fixed capacity array.

To test whether `SmallVec` could be used for verification of Rust programs using `SeaHorn`, I created verification jobs for each of the methods.

## Limitations

Since the fixed capacity of `SmallVec` is constant and must be known at compile time, it isn't possible to verify the class for non-deterministic capacities. In addition, `SeaHorn` takes a long time to run for certain tests, so I had to limit the capacity to very small values, as low as 2, in some cases.

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

For several tests, I was facing issues where `SeaHorn` was not reaching certain points in the code. After debugging this issue, I determined that it was being caused by loops not unrolling completely. For most of the tests, I push non-deterministic values to the vector in a loop to initialize it for testing. All of the tests using loops like this were only having one iteration unrolled, essentially limiting the length of the vector to 1. I was able to fix this issue by using the `bound` option and setting it to whatever capacity was being used for the vector in that specific case.

Another issue that I faced was that `SeaHorn` was having issues with verification whenever the vectors went beyond their fixed capacity and required more memory allocated on the heap. Even if only one additional space in a vector is required, a very large `alloca` expression is generated, causing `SeaHorn` to take excessive time to get through this stage and in many cases failing. Through debugging, I determined that the expressions being generated were so large that my computer was running out of memory when trying to print them. This issue was fixed by limiting the size of expressions that can be printed and printing a placeholder string whenever they are over the limit. This fix can be found [here](https://github.com/seahorn/seahorn/pull/498).

## Source Code

Full source code for the tests can be found [here](https://github.com/thomashart17/c-rust/tree/main/src/rust-jobs) under the 4 directories whose names start with: "smallvec". Note that the tests are split into 4 separate jobs to allow for different `bound` settings required by certain tests.
