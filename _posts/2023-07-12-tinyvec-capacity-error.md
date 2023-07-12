---
layout: post
title:  "Rust TinyVec Capacity Error"
subtitle: "Missing limit to array capacity"
date:   2023-07-10 15:00:00
categories: [seahorn, rust, verification]
---

In Rust's `tinyvec` crate, there was an error present that caused allowed for the capacity of a vector to be larger than the largest possible length. This error was fixed in [this](https://github.com/Lokathor/tinyvec/commit/48c004ddc27eebc8484a30602c894ba3915ec721) commit. I created a verification job to test the previous version of this code for the error and to show that `SeaHorn` would catch it.

## Verification Code

```rust
pub extern "C" fn entrypt() {
    let vec: ArrayVec<[u32; u16::MAX as usize + 1]> = ArrayVec::new();

    sea::sassert!(vec.capacity() == u16::MAX as usize);
}
```

Since `u16::MAX` should be the largest possible capacity for the array, this assertion should be true. However, since I used the older version of the crate, this is not the case and the assertion should fail.

## Issues

When running `SeaHorn`, it struggled to run for large array sizes and this test was exiting with an exit code of 137 from `SeaHorn` without returning a value of `sat` or `unsat`. After looking into it, I determined that the issue was happening because `SeaHorn` was running out of memory. This was being caused by the fat pointer pass adding a ton of extra lines to the code. It went from ~1200 lines in the previous stage to over 5 million lines. This issue does not appear when running `SeaHorn` with `bpf` instead of `fpf`.
