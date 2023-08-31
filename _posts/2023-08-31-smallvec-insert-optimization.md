---
layout: post
title:  "Rust SmallVec Insert Optimization"
subtitle: "Optimization using lots of unsafe code."
date:   2023-08-31 15:30:00
categories: [seahorn, rust, verification]
---

In Rust's `smallvec` crate, there was an optimization done to the `insert` method using a lot of unsafe code in [this](https://github.com/servo/rust-smallvec/commit/b2335682bcaf6d0b33d6c0caeb077d9aaa608b6d) commit. The optimization was done to eliminate calls to `copy` when insertion was at the end of the vector. I created a verification job to verify that this did not introduce any issues.

## Verification Code

```rust
pub extern "C" fn entrypt() {
    let mut v: SmallVec<[u32; 8]> = SmallVec::new();

    let len: usize = sea::nd_usize();
    sea::assume(len < 8);

    for _i in 0..len {
        v.push(sea::nd_u32());
    }

    let insert_point: usize = sea::nd_usize();
    sea::assume(insert_point <= len);

    v.insert(insert_point, sea::nd_u32());

    if insert_point == len {
        sea::sassert!(v.copied() == false);
    } else {
        sea::sassert!(v.copied() == true);
    }

    sea::sassert!(v.len() == len + 1);
}
```

I slightly modified the `SmallVec` source code to include a `copied` flag that tracks whether the vector has been copied or not. This flag is set to `true` when the vector is copied and `false` when it is not. This test case pushes non-deterministic values to a vector and then inserts a value at a random index. If the index is at the end of the vector, then the vector should not be copied, but if it is not, then the vector should be copied. This test case worked pretty much right away without any significant issues.
