---
layout: post
title:  "Rust SmallVec Drain Error"
subtitle: "Panic on arithmetic overflow."
date:   2023-08-31 15:30:00
categories: [seahorn, rust, verification]
---

In Rust's `smallvec` crate, there was an issue present in the `drain` method where arithmethic overflows were occurring but not panicking. This error was fixed in [this](https://github.com/servo/rust-smallvec/pull/259) pull request. I created a verification job to show that `SeaHorn` would catch this error.

## Verification Code

```rust
pub extern "C" fn entrypt() {
    const CAP: usize = 8;
    let mut v: SmallVec<[u8; CAP]> = SmallVec::new();

    let len: usize = sea::nd_usize();
    sea::assume(len <= CAP);

    for _i in 0..len {
        v.push(sea::nd_u8());
    }

    // In smallvec versions 1.6.0 and below, this will execute succesfully.
    // In newer versions, this will panic.
    let _ = v.drain(..=usize::MAX);

    // This assertion will only be reached if the previous operation doesn't panic.
    sea::sassert!(false);
}
```

This test case pushes non-deterministic values to a vector and then calls drain on that vector with an upper bound of `usize::MAX`. This should cause an arithmetic overflow and panic, but in older versions of the crate, it does not. The assertion at the end of the test case should not be reached if the panic occurs.

## Issues

This issue was not always present depending on compilation options set in the `Crago.toml` file regarding overflow checks. To make sure that the issue was always present, I added the following lines to the workspace `Cargo.toml` file:

```toml
[profile.dev.package.smallvec-drain-error-lib]
overflow-checks = false

[profile.release.package.smallvec-drain-error-lib]
overflow-checks = false
```

This disables checks for overflow by the compiler and makes sure that the issue is always present.
