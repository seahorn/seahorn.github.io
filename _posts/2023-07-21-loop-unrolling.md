# Issues With Loop Unrolling in SeaHorn When Using Drop in Rust

SeaHorn is an automated analysis tool used to verify weather a program will run within a set of constraints defined by what are called assertions. When a constraint is violated the program will return sat, meaning a counterexample has been found/satisfied (bad), otherwise it will give unsat meaning no counter example could be found/satisfied (good). When analyzing code, SeaHorn will actually look at the LLVM intermediate representation (IR, a language similar to assembly) generated from compiling the code. This allows for SeaHorn to run jobs on any language compiled with LLVM.

Here's an example:
``` Rust
let mut v: Vec[i32];
...
sea::sassert!(v.len() < 20);
```
In this case, if length is less that 20, the assertion passes; if not, the assertion fails and we get sat and a counterexample showing what trace causes the assertion to fail.


Where issues arise is when loops are used, as SeaHorn cannot directly analyze them. The solution is to unroll them.

For example:
``` Rust
for i in 0..2 {
    sea::sassert!(i < 10);
}
// becomes ...
let i = 0;
sea::sassert!(i < 10);
i =+ 1;
sea::sassert!(i < 10);
```

Many small loops will automatically get unrolled by the complier, but in certain cases such as when loops are too large, or when they depend on complex parameters they don't get unrolled and SeaHorn can't analyze the program. What this ends up looking like in practice is what ever code (and assertions) comes after the loop doesn't get analyzed, and as long as the all assertions prior to the loop are valid, SeaHorn returns unsat despite there potentially being false assertions later on.


## Example

During the implementation of a custom vector as described in [The Rustonomicon](https://doc.rust-lang.org/nomicon/vec/vec.html), SeaHorn in certain cases was not able to effectively analyze loops, particularly those found in the Drop traits of the vector. Here's what the code looks like (don't worry about the vectors own deallocation which is handled elsewhere).

``` Rust
impl<T> Drop for CustomVec<T> {
    fn drop(&mut self) {
        while let Some(_) = self.pop() { }
    }
}

```
The code simply pops each element of the vector to ensure that any droppable types in the vector also get dropped themselves.
What ends up happening is the compiler, for it's own reasons, chooses not to unroll this loop, meaning SeaHorn cannot analyze further.

Interestingly though, both of theses implementations of drop (which all do the same thing at the end) work fine (get unrolled)

``` Rust
impl<T> Drop for CustomVec<T> {
    fn drop(&mut self) {
        sea::sea_printf!("Print Something Before Loop");
        while let Some(_) = self.pop() {
            sea::sea_printf!("Print Something Before Loop");
        }
    }
}
```
``` Rust
impl<T> Drop for CustomVec<T> {
    fn drop(&mut self) {
        unsafe {
            let slice: &mut [T] = core::slice::from_raw_parts_mut(self.ptr(), self.len);
            _ = Box::from_raw(copy_slice as *mut [T]);
        }
    }
}
```

In the first example, the exact same code is used, with redundant print statements added. In the second example, we create a slice giving us a window into the vector, and then take ownership of the values in that slice using ```Box::from_raw()```, such that when it goes out of scope, drop is called on each element.


What's simple about these examples is that they work. What's confusing is why?

## What's happening?

Compilation happens in three main stages: converting the code into LLVM IR, and then optimizing that code (sometimes in many stages of its own). Then converting that optimized intermediate representation into machine code which can be read by you processor. One of these optimizations is loop unrolling, but it comes at a cost. When a compiler unrolls a loop the runtime of that loop is generally faster, but when loops are large the unrolling process makes the binary file larger, so theres a tradeoff. Additionally, the compiler may not be able to unroll the loop, and will instead leave it as is. It's important to note that different compilers behave differently, therefore the results won't always be the same. In the fist case, our compiler decided to not unroll the loop, but the last 2 examples it did unroll them as for what ever reason, it felt that doing so would lead to more optimal code at minimal cost, which is why they both work.

## How to Catch the Issue

These issues can be deceiving, as when they come up SeaHorn won't analyze the code or any code after, meaning as long as all the assertions prior to the issue pass you will get unsat making it hard to catch. To check weather this is happening you should put a false assertion in some segment of code you know will certainly and only get called after where the issue may occur. If you get unsat, despite there being a false assertion, you know that somewhere prior SeaHorn stopped analyzing the code. A good place to start would be at the end of your main function, and to then use a binary search to pinpoint where the flaw occurs, and keep doing so until you find the fine line between where SeaHorn analyzes, and doesn't analyze you false assertion.

For example assume that a program you have written gives ```unsat```:
``` Rust
fn main() {
    f1();
    f2();
    f3();
    f4();
}
```
Place an assertion that will definitely get called and fail
``` Rust
fn main() {
    f1();
    f2();
    f3();
    f4();
    sea::sassert!(false);
}
```
This should give ```sat```, as sea::sassert!(false) will always fail, but if it passes (gives ```unsat```), somewhere in the code SeaHorn stopped analyzing. So then cut the code in half.
``` Rust
fn main() {
    f1();
    f2();
    sea::sassert!(false);
    f3();
    f4();
    // sea::sassert!(false);
}
```
If this gives ```sat```, then the code up to f3() gets analyzed. Cut in half again.
``` Rust
fn main() {
    f1();
    f2();
    // sea::sassert!(false);
    f3();
    sea::sassert!(false);
    f4();
    // sea::sassert!(false);
}
```
If we now get ```sat```, we can confidently say tha the code stop analyzing somewhere in f4(), as the code analyzes past f3(), but  not past f4(). 
If we get ```unsat```, then an issue must lie in f3() as from the previous checks, all the code before f3() got analyzed.
Now repeat this process inside the function until you find the culprit.
Note that this issue can appear in many blocks of code, so just because there's in issue in f3(), doesn't mean there isn't one in f4().

Another more simple method, would be to just jump to a loop which you think may be causing the issue and to check it and so on.


## Solution

There are two main ways to solve this issue, both involve getting the loop unrolled.

The first is the one shown above: trail and error for coming up with different ways to perform the same task, until the compiler finds it optimal to unroll your loop. While this way is possible, is not optimal, as there's no concrete rule to when your loop will get unrolled, and such behavior will vary depending on the compiler being used, and weather they change their loop unrolling optimizations in an update.

A better solution is to use the bound flag when calling the verify script
``` shell
./verify source/job --bound=20
```
What this will do is unroll any loops which the compiler doesn't unroll on its own 20 times. Therefore, it's important to make sure that none of your loops run more that 20 times in order to get an equivalent program. It's important to note that this will not make all you loops run 20 times, as before each iteration an if condition will be checked to determine weather to continue. Additionally, this number doesn't have to be 20, and can be adjusted to whatever specifications are needed.

By bounding your loops, you can ensure that SeaHorn will correctly analyze the loops in your program. Just remember that the bound should always be greater than or equal to the maximum iterations you would expect in all the loops coming out of your program. Note that as this feature has time limitations, as the higher the bound the more code SeaHorn needs to analyze. Therefore you should aim for the lowest upper bound you can find, and in certain cases, you may need to adjust you program to get a lower bound as this feature sometimes works for bounds in the hundreds, but in other cases, only for bounds up to 40, 20, or even less.

It's important to exercise caution when testing loops with SeaHorn and to keep a close eye on its behavior as the tool is imperfect, but with proper care, could be made ever useful.

## Try it Yourself

Due to the significance of this issue, we've created a job in out Rust tool to allow for you to test test this issue.

Go to our [Rust Repo](https://github.com/thomashart17/c-rust) and play around with the rust job ```custom-vec-loop-unroll```. Remember to first have [SeaHorn](https://github.com/seahorn/seahorn/tree/main) set up.

``` Rust
impl<T> Drop for CustomVec<T> {
    fn drop(&mut self) {
        // Version 1: Remember to update Drop for RawVec
        // This should fail (always give unsat), add unroll bound to fix
        while let Some(_) = self.pop() { }


        // Version 2: Remember to update Drop for RawVec
        // This should work without the unroll bound, up to a limit
        // sea::sea_printf!("Before Loop");
        // while let Some(_) = self.pop() {
            // sea::sea_printf!("During Loop");
        // }


        // Version 3: Remember to update Drop for RawVec
        // This should work without the unroll bound, up to a limit
        // Box::from_raw also handles the deallocation of the vector itself
        // unsafe {
        //     let slice: &mut [T] = core::slice::from_raw_parts_mut(self.ptr(), self.len);
        //     _ = Box::from_raw(copy_slice as *mut [T]);
        // }


        // Try Your Own Version
        // Here
    }
}
```

To build after making changes
``` shell
ninja
```

To run without loop bound
``` shell
./verify src/rust-jobs/custom-vec-final
```
To run with loop bound
``` shell
./verify src/rust-jobs/custom-vec-final --bound=100
```
