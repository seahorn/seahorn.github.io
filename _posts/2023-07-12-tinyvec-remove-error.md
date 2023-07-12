---
layout: post
title:  "Rust TinyVec Remove Error"
subtitle: "No panics on invalid indices"
date:   2023-07-12 15:30:00
categories: [seahorn, rust, verification]
---

In the `tinyvec::ArrayVec::remove` method, there was an error present where the method would not panic if the provided index was out of bounds. This error was fixed in [this](https://github.com/Lokathor/tinyvec/pull/29/commits/fd3c92c35109a4b025738fe71bb0fd739c3d6002) commit. I created a verification job to test the previous version and to show that `SeaHorn` is capable of finding it.

## Verification Code

```rust
pub extern "C" fn entrypt() {
    let mut v: ArrayVec<[u32; 8]> = ArrayVec::new();
    let len: usize = sea::nd_usize();
    sea::assume(len <= 8);

    for _i in 0..len {
        v.push(sea::nd_u32());
    }

    let remove_point: usize = sea::nd_usize();
    sea::assume(remove_point <= 8);

    v.remove(remove_point);

    if remove_point < len {
        sea::sassert!(v.len() == len - 1);
    } else {
        // If remove_point is out of bounds, then the call to remove should panic and this assertion should not be reachable.
        sea::sassert!(false);
    }
}
```

Since the method should panic if `remove_point` is greater than `len`, the `sea::sassert!(false)` should not be reached and the test will fail if the call to `remove` does not panic. This test worked pretty much right away without any significant issues. When testing with `tinyvec` version 0.2.0, the last release before the error was fixed, the assertion was reached, showing that the method did not panic. Using later versions of the crate, the assertion is never reached, showing that the issue was fixed.
