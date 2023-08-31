---
layout: post
title:  "Rust SmallVec Memory Leak"
subtitle: "Leaking memory on panicking insert_many"
date:   2023-08-31 15:30:00
categories: [seahorn, rust, verification]
---

In Rust's `smallvec` crate, there was a memory leak present in one of the test cases for the `insert_many` function. The error would trigger if there was a call to `panic` in the iterator provided to the method. This issue was fixed in [this](https://github.com/servo/rust-smallvec/pull/213/commits/8ddf61330d73bd1b33ed01e03f2bf0b8aaba8d11) commit. I created a verification job to show that `SeaHorn` would catch this error.

## Verification Code

This code was modified from the test case in the `smallvec` crate to work with `SeaHorn`.

```rust
pub extern "C" fn entrypt() {
    struct PanicOnDoubleDrop {
        dropped: Box<bool>,
    }

    impl Drop for PanicOnDoubleDrop {
        fn drop(&mut self) {
            assert!(!*self.dropped, "already dropped");
            *self.dropped = true;
        }
    }

    struct BadIter;
    impl Iterator for BadIter {
        type Item = PanicOnDoubleDrop;
        fn size_hint(&self) -> (usize, Option<usize>) {
            (1, None)
        }
        fn next(&mut self) -> Option<Self::Item> {
            panic!()
        }
    }

    let mut vec: SmallVec<[PanicOnDoubleDrop; 0]> = smallvec![
        PanicOnDoubleDrop {
            dropped: Box::new(false),
        },
        PanicOnDoubleDrop {
            dropped: Box::new(false),
        },
    ];
    let result = ::std::panic::catch_unwind(move || {
        vec.insert_many(0, BadIter);
    });
}
```

The panicking iterator causes the `insert_many` method to leak memory, which `SeaHorn` should be able to catch.

## Issues

In our repository, we are using custom panic handlers to define our own behavior for panics. As this job requires a panic to occur and then be caught, we don't wan't to immediately exit or call `verifier_error` when it occurs, like we were doing before. To fix this issue, I updated our `cmake` configuration to include an option that allowed us to disable the custom panic handler and use the default features that allow panics to unwind. After adding this option, the job worked correctly and `SeaHorn` was able to catch the leak.
