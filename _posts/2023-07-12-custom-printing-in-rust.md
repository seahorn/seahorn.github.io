---
layout: post
title:  "Custom Print Macros in Rust"
subtitle: "the custom_print crate"
date:   2023-07-12 13:30:00
categories: [seahorn, rust]
---

When using `SeaHorn` to verify Rust programs, the standard library print macros can cause increased runtime and can sometimes cause errors. Most of the time, the print macros aren't the focus of our verification jobs so we wanted to elimiate this issue. To accomplish this, we used the [custom_print](https://docs.rs/custom-print/latest/custom_print/) crate to define custom print macros void of any functionality and replace the standard library print macros with them.

## Code

Using `custom_print` version 1.0.0, we were able to define the custom print macros as follows:

```rust
pub struct NullWriter;

impl NullWriter {
    pub fn write_fmt(&mut self, _: Arguments<'_>) -> fmt::Result { Ok(()) }
}

custom_print::define_macros!(
    #[macro_export]
    {print, println, eprint, eprintln},
    sea::print_macros::NullWriter
);
```

All print macros require a `Writer` in order to function properly, however the default writer classes were increasing runtimes in `SeaHorn` so we defined our own `NullWriter` class. As the name would suggest, this class takes in arguments and does nothing with them, which improves the runtime of `SeaHorn`. After defining our own `Writer` class, we were then able to define the macros using the `custom_print::define_macros` macro. By giving the print macros the same names as the ones in the standard library, and working in a `#![no_std]` environment, we are able to overwrite the default macros.

## Usage in Other Crates

The `#[macro_export]` line and the namespace on `NullWriter` are required to be able use these macros in other crates within a workspace. If the macros were only for use in the same crate, these would not be necessary.

## Alternatives

While getting these macros to work, we tested several alternatives to replacing the standard print macros. We worked based on the examples provided by the `custom_print` [documentation](https://docs.rs/custom-print/latest/custom_print/). Although some of these examples did improve runtimes, none of them were as efficient as using our custom `Writer` implementation.

Source code for these alternatives can be found in the jobs with `custom-print` in their names [here](https://github.com/thomashart17/c-rust/tree/main/src/rust-jobs).
