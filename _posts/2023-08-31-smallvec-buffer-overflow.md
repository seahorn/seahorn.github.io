---
layout: post
title:  "Rust SmallVec Buffer Overflow"
subtitle: "Buffer overflow in insert_many"
date:   2023-08-31 15:30:00
categories: [seahorn, rust, verification]
---

In Rust's `smallvec` crate, there was a buffer overflow present in the `insert_many` function. This [error](https://github.com/servo/rust-smallvec/issues/252) was occuring when the iterator contained more items than the lower bound of the `size_hint`. Using the reproduction code from the issue, I was able to create a verification job to show that `SeaHorn` would catch this error.

## Verification Code

This code was modified from the reproduction code in the pull request for this [error](https://github.com/servo/rust-smallvec/issues/252).

```rust
pub extern "C" fn entrypt() {
    let mut v: SmallVec<[u8; 0]> = SmallVec::new();

    // Spill on heap
    v.push(sea::nd_u8());

    // Allocate string on heap
    let s = String::from("A");

    // Prepare an iterator with small lower bound
    let iter = (0u8..=3).filter(|n| n % 2 == 0);

    // Triggering the bug
    v.insert_many(0, iter);

    // Uh oh, heap overflow made smallvec and string to overlap
    sea::sassert!(!v.as_ptr_range().contains(&s.as_ptr()));
}
```

Using this test case, I was able to show that `SeaHorn` would catch the buffer overflow and that it was fixed in newer versions of the crate.

## Issues

While creating this verification job, I was getting many of these error messages from `SeaHorn`:

```
Error: unsupported memcpy due to size and/or alignment.
```

This error was fixed by using the `horn-bv2-word-size` option set to 1, making `SeaHorn` use 8-bit words.
