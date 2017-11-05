---
layout: post
title:  "Proving Memory Safety with SeaHorn"
subtitle: "how to automatically prove memory safety?"
date:   2017-05-27 21:24:45
categories: [seahorn, memory safety]
---

A program is *memory safe* if the following memory access errors never occur:

* buffer overflow
* null pointer dereference
* user after free
* use of uninitialized memory
* illegal free (of an already-freed or non-allocated pointer)

In this post, we will focus only on buffer overflows. A *buffer
overflow* occurs when a program while reading or writing from/to a
buffer, access outside the boundaries out-of-bound of the buffer. The
*Array Bounds Checking* (ABC) problem focuses on whether there is a
pointer that can never access to a memory address which has not been
previously allocated. In the rest of this post, we will show how we
can use SeaHorn to tackle the ABC problem.

Consider we want to prove that all the array accesses are in-bounds
for the next example `abc1.c`:

{% highlight c linenos=table %}
#include "seahorn.h"
extern int nd();
#define N 10
int main(int argc, char**argv) {
  int i;
  int a[N];
  for (i = 0; i < N; i++)  {
    a[i] = i;
  }

#ifndef ERROR
  printf("%d\n", a[i-1]);
#else
  printf("%d\n", a[i]);
#endif   
  return 0;
}
{% endhighlight %}

To do that, we can instrument `abc1.c` by adding two ghost variables
`offset` and `size` per each pointer `p`. The variable `offset` is the
distance from the base address of the memory object that contains `p`
to the actual `p`, and `size` is the actual size of the allocated
memory for `p`. In addition, we need to instrument each array access
to update `offset` accordingly while adding the in-bound assertions
that we would like to prove. We call this instrumented program
`abc1_inst.c`:

{% highlight c linenos=table %}
#include "seahorn.h"
extern int nd();

#define N 10

int main(int argc, char**argv)
{
  int i;
  int a[N];

  // ghost variables
  int offset;
  unsigned int size;

  // initialization of the ghost variables
  offset = 0;
  size = sizeof(a);

  for (i = 0; i < N; i++) {
    // update offset
    offset = sizeof(int)*i;
	// check the array write is in-bounds
    sassert (offset < size);
    sassert (offset >= 0);

    a[i] = i;
  }

#ifndef ERROR
  // update offset
  offset = sizeof(int)*(i-1);  
  // check the array read is in-bounds
  sassert (offset < size);
  sassert (offset >= 0);

  printf("%d\n", a[i-1]);
#else
  // update offset
  offset = sizeof(int)*i;  
  // check the array read is in-bounds
  sassert (offset < size);
  sassert (offset >= 0);

  printf("%d\n", a[i]);
#endif   
  return 0;
}

{% endhighlight %}

Now, we are ready to prove `abc1_inst.c`:

{% highlight c %}
sea pf -O0 abc1_inst.c --show-invars
{% endhighlight %}

SeaHorn can prove the program is safe! (i.e., all array accesses are
in-bounds). The invariants produced by SeaHorn are:

{% highlight c %}
unsat
Function: sassert
sassert@_call: true
sassert@_ret: true
Function: main
main@entry: true
main@_bb:
	(main@%i.0.i>=0)
	(main@%i.0.i<=10)
main@verifier.error.split: false
{% endhighlight %}

Let us also try a buggy version where the array index `i` is accessed
when `i=10`:

{% highlight c %}
sea pf -O0 abc1_inst.c -DERROR --show-invars
{% endhighlight %}

SeaHorn finds a counterexample of length 10.

As you can see, this instrumentation can be done manually in a
straightforward manner if there is not many pointers in the
program. Otherwise, it can be a very tedious process. Apart from that,
the instrumentation can get really complicated with functions and
specially when pointers are stored in memory.

Consider another program called `abc2.c` similar to `abc1.c` but where
two arrays `a` and `b` are initialized instead of one:

{% highlight c linenos=table %}
#include "seahorn.h"
extern int nd();

#define N 10

int main(int argc, char**argv) {

  int8_t a[N];
  int8_t b[N];

  int i;
  for (i = 0; i < N; i++) {
    a[i] = i;
  }

  int j;
  for (j = 0; j < N; j++) {
    b[j] = j;
  }

#ifndef ERROR
  printf("%d\n", a[i-1]);
  printf("%d\n", b[i-1]);  
#else
  printf("%d\n", b[i]);    
#endif

  return 0;
}
{% endhighlight %}

We propose now a simpler instrumentation that does not need to
instrument each pointer individually. The idea is to choose
*non-deterministically* a pointer and keep track of the base address
and size of the region to which the pointer points to. The key insight
is that the solver takes care of resolving the non-determinism. The
instrumentation adds only four global variables:

* base: non-deterministic (positive) base address of a non-empty
  allocation site.
* size: size of the allocated region.
* ptr: a valid address between [base,...,base+size)
* offset: numerical difference between ptr and base.

The key advantage is that we keep track of only four variables
regardless of the number of pointers in the program. Moreover, it is
really easy to instrument pointers that are stored in memory. We show
the instrumented program called `abc2_inst.c` following this global
encoding:

{% highlight c linenos=table %}
#include "seahorn.h"
extern int nd();

#define N 10

typedef ptrdiff_t sea_ptrdiff_t;
typedef ptrdiff_t sea_size_t;

extern int nd(void);
extern int8_t *nd_int8_ptr (void);
extern sea_size_t nd_sea_size_t (void);

static int8_t* base;
static int8_t* ptr;
static sea_ptrdiff_t offset;
static sea_size_t size;

int main(int argc, char**argv)
{

  // -- init
  base = nd_int8_ptr ();
  assume (base > 0);
  size = nd_sea_size_t ();
  assume (size >= 0);
  ptr = 0;  
  offset = 0;

  int8_t a[N];

  // -- allocation of a
  int8_t *b_a  = &(a[0]);  
  if (!ptr  && (b_a == base)) {
    // ptr = base;
    ptr = nd_int8_ptr ();
    assume (ptr == base);
    assume (size == sizeof(int8_t) * N);
    offset = 0;
  } else {
    assume (base+size < b_a);
  }

  int i=0;
  int8_t* q_a = b_a;
  for (; i < N; i++, q_a++) {
    // -- pointer arithmetic
    if (nd() && (q_a == ptr)) {
      ptr = nd_int8_ptr ();
      assume (ptr == (q_a + 1));
      offset += 1;
    }
    // -- a[i] = i;
    *q_a = i;
  }

  int8_t b[N];
  int8_t *b_b  = &(b[0]);

  // -- allocation of b
  if (!ptr  && (b_b == base)) {
    // ptr = base;
    ptr = nd_int8_ptr ();
    assume (ptr == base);
    assume (size == sizeof(int8_t) * N);
    offset = 0;
  } else {
    assume (base+size < b_b);
  }

  int j=0;
  int8_t* q_b = b_b;
  for (; j < N; j++, q_b++) {
    // -- pointer arithmetic
    if (nd() && (q_b == ptr)) {
      ptr = nd_int8_ptr ();
      assume (ptr == (q_b + 1));
      offset += 1;
    }
    // -- b[j] = j;
    *q_b = j;
  }

#ifndef ERROR
  // -- safe memory access
  if (ptr && (ptr  == (b_a + (i-1)))) {
    sassert (offset < size);
  }
  printf("%d\n", a[i-1]);

  // -- safe memory access
  if (ptr && (ptr  == (b_b + (j-1)))) {
    sassert (offset < size);
  }
  printf("%d\n", b[j-1]);

#else
  // -- safe memory access
  if (ptr && (ptr  == (b_a + (i-1)))) {
    sassert (offset < size);
  }
  printf("%d\n", a[i-1]);

  // -- unsafe memory access
  if (ptr && (ptr  == (b_b + j))) {
    sassert (offset < size);
  }
  printf("%d\n", b[j]);

#endif   
  return 0;
}
{% endhighlight %}

We can now prove that `abc2_inst.c` is safe by running the command:

{% highlight c %}
sea pf -O0 abc2_inst.c --show-invars
{% endhighlight %}

We do not show the invariants but we recommend you to run this command
and look at the invariants to convince yourself why the program is
safe.

We can also check that the buggy version is indeed unsafe by running:

{% highlight c %}
sea pf -O0 abc2_inst.c --show-invars -DERROR
{% endhighlight %}

We will not show either the counterexample but if you have read
previous posts then you should be able to generate a harness program
and find out why the program is unsafe using your favorite debugger.

So we have seen that instrumenting a program to prove absence of
buffer overflows it is a lot of fun. But if you do not want to
instrument manually your program, SeaHorn can add fully automatically
the array bounds checks. For instance, if you execute the command:

{% highlight c %}
sea pf -O0 abc2.c --abc=global --show-invars
{% endhighlight %}

then you should see the following output:

{% highlight c %}
unsat
Function: main
main@entry: true
main@_i.0.i:
		(main@%i.0.i>=0)
	(main@%i.0.i<=5)
main@_j.0.i:
		(main@%j.0.i>=0)
	(main@%i.0.i>=5)
	(main@%i.0.i<=5)
main@verifier.error.split: false
{% endhighlight %}

SeaHorn proves absence of buffer overflows by typing one single
command!

Well, this is the end of our post where we have shown two different
ways of instrumenting a program to prove absence of buffer
overflows. Luckily, SeaHorn can add the instrumentation automatically
so that we do not need to do it by ourselves.
