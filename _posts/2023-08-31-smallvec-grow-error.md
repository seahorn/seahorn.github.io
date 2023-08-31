---
layout: post
title:  "Rust SmallVec Grow Error"
subtitle: "Incorrect shrinking of spilled vector."
date:   2023-08-31 15:30:00
categories: [seahorn, rust, verification]
---

In Rust's `smallvec` crate, there was an issue present in the `grow` method where the vector was not correctly shrinking after it had been spilled. This issue was fixed in [this](https://github.com/servo/rust-smallvec/commit/50a50ed90d6ad78d812a40680257d8338843869a) commit. I created a verification job to show that `SeaHorn` would catch this error.

## Verification Code

```rust
pub extern "C" fn entrypt() {
    let mut v: SmallVec<[u32; 8]> = SmallVec::new();
    
    let len: usize = sea::nd_usize();
    sea::assume(len <= 16);

    for _ in 0..len {
        v.push(sea::nd_u32());
    }

    v.clear();

    // Shrink to inline.
    v.grow(8);

    sea::sassert!(!v.spilled());
    sea::sassert!(v.capacity() == 8);
    sea::sassert!(v.len() == 0);
}
```

This test case pushes non-deterministic values to a vector, clears it, then calls grow to shrink the vector back to it's original inline capacity. This should cause the vector to shrink back to it's original capacity and not be spilled, but in older versions of the crate, it does not. I did not face any significant issues while writing this test case and it worked pretty much right away.
