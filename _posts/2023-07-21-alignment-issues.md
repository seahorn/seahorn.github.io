# Catching Alignment Issues in Rust using SeaHorn


SeaHorn is an automated analysis framework for LLVM-bases languages, which includes Rust. Given a set of constraints, it can be used to prove weather a givin program will stay within such bounds, and if not, it can find counter examples and detail which constraints get violated. One such constraint a user may want to invoke, would be to ensure that the pointers in a program remain aligned throughout its lifespan.


## What are Alignment Issues and Why Care

Alignment issues in programming arise when data objects are not properly aligned in memory according to the requirements of the underlying hardware architecture. For example on a 32 bit processor a 4 byte/32bit integer starting at address 0x03 would not be aligned, as it lives partly in two 4 byte blocks. Most processors have alignment requirements for accessing certain types of data. When these requirements are not met, it can result in performance penalties, your program crashing, or a hardware exception.

With regards to the latter two, these potential issues are usually ironed out by the compiler, leading to former.

If your hardware is unable to handle unaligned pointers, your compiler will generate code to check for and ensure proper alignment, which will leads to performance issues. If the hardware your using allows for the use of unaligned pointers, while you code may run fine, there will still be performance penalty. Additionally, even if your hardware allows for this, the compiler may still introduce its own alignment checks and changes. In any case, having unaligned pointers in your program at best leads to a slower run time.

## Example

In the [Rustonomicon](https://doc.rust-lang.org/nomicon/vec/vec.html) an alignment issue is encountered during the implementation of a custom vector which can handle zero sized types (ZST). When using a double ended iterator, the author uses a standard buffer with a start and end pointer, and moves them along in chunks when iterating. But, when a ZST is used, there are no start and end pointers, nor can you increment/decrement those pointers as the data size is zero. So what do you do?

The solution they use is to use an arbitrary start pointer, and an end pointer equal to the start plus the vectors length, and to increment the pointer by one like a counter. Dereferencing shouldn't be an issue as the address of a ZST doesn't matter since it takes up no memory.

But wait! There's an issue, can you spot it?

``` Rust
impl<T> RawValIter<T> {
    unsafe fn new(slice: &[T]) -> Self {
        RawValIter {
            start: slice.as_ptr(),
            end: if mem::size_of::<T>() == 0 {
                ((slice.as_ptr() as usize) + slice.len()) as *const _
            } else if slice.len() == 0 {
                slice.as_ptr()
            } else {
                slice.as_ptr().add(slice.len())
            },
        }
    }
    fn next(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                let result = ptr::read(self.start);
                self.start = if mem::size_of::<T>() == 0 {
                    (self.start as usize + 1) as *const _
                } else {
                    self.start.offset(1)
                };
                Some(result)
            }
        }
    }
}
```

When incrementing by 1 for zero sized types, we end up performing unaligned pointer reads. The one thing we should avoid.

## Fix

While the the ptr::read function is a noop for ZSTs, it's best to still keep them aligned to avoid any compiler checks for our use of unaligned pointers. What we instead do is use ```NonNull::<T>::dangling()``` which will always give us an aligned pointer. All while keeping our counter untouched.

``` Rust
fn next(&mut self) -> Option<T> {
    if self.start == self.end {
        None
    } else {
        unsafe {
            if mem::size_of::<T>() == 0 {
                self.start = (self.start as usize + 1) as *const _;
                Some(ptr::read(NonNull::<T>::dangling().as_ptr()))
            } else {
                let old_ptr = self.start;
                self.start = self.start.offset(1);
                Some(ptr::read(old_ptr))
            }
        }
    }
}
```

## Not Enough

Despite solving the issue of reading unaligned pointers, the issue of having potentially unaligned pointers in our program still remains. We shouldn't hope that the compiler sees that only aligned pointers are read from, and that our unaligned pointers are just being used as counters. Instead we should aim to have our program never have any unaligned pointers, to avoid any possible mix up leading the compiler to produce sub optimal code.

The solution is simple, instead of incrementing by 1, increment by the alignment of the pointer. This will ensure our compiler produces an optimal binary, and that we ourselves don't run into any later issues when using these pointers.

``` Rust
impl<T> RawValIter<T> {
    unsafe fn new(slice: &[T]) -> Self {
        RawValIter {
            start: slice.as_ptr(),
            end: if mem::size_of::<T>() == 0 {
                ((slice.as_ptr() as usize) + slice.len()*mem::align_of::<u32>()) as *const _
            } else if slice.len() == 0 {
                slice.as_ptr()
            } else {
                slice.as_ptr().add(slice.len())
            },
        }
    }
    fn next(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                if mem::size_of::<T>() == 0 {
                    self.start = (self.start as usize + mem::align_of::<u32>()) as *const _;
                    Some(ptr::read(NonNull::<T>::dangling().as_ptr()))
                } else {
                    let old_ptr = self.start;
                    self.start = self.start.offset(1);
                    Some(ptr::read(old_ptr))
                }
            }
        }
    }
}
```

## SeaHorn

Using SeaHorn we can catch these sorts of issues to ensure our pointers always remain aligned. To catch such error we can use our assertion tool as shown below. When an assertion fails, SeaHorn will give a trace showing how and why it failed.

``` Rust
``` Rust
impl<T> RawValIter<T> {
    unsafe fn new(slice: &[T]) -> Self {
        // our check
        sea::sassert!((slice.as_ptr() as usize) % mem::align_of::<T>() == 0);

        RawValIter {
            start: slice.as_ptr(),
            end: if mem::size_of::<T>() == 0 {
                ((slice.as_ptr() as usize) + slice.len()*mem::align_of::<T>()) as *const _
            } else if slice.len() == 0 {
                slice.as_ptr()
            } else {
                slice.as_ptr().add(slice.len())
            },
        }
    }
    fn next(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {s
                if mem::size_of::<T>() == 0 {
                    self.start = (self.start as usize + mem::align_of::<T>()) as *const _;
                    // our check
                    sea::sassert!((self.start as usize) % mem::align_of::<T>() == 0);

                    Some(ptr::read(NonNull::<T>::dangling().as_ptr()))
                } else {
                    let old_ptr = self.start;
                    self.start = self.start.offset(1);
                    // our check
                    sea::sassert!((self.start as usize) % mem::align_of::<T>() == 0);

                    Some(ptr::read(old_ptr))
                }
            }
        }
    }
}
```

We can now write whatever test cases we want for our vector, and using these assertions prove that our pointers will never get unaligned.

You can try this yourself by going to our [Rust Repo](https://github.com/thomashart17/c-rust) and play around with the rust job ```custom-vec-job-to-specify```. Remember to first have [SeaHorn](https://github.com/seahorn/seahorn/tree/main) set up.