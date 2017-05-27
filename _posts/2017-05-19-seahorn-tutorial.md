---
layout: post
title:  "SeaHorn Tutorial"
subtitle: "Basic examples and how to prove memory safety"
date:   2017-05-20 14:56:45
categories: [seahorn, usage, memory safety]
---

It has been a while since our last blog but here we are again! We
recently gave a tutorial about SeaHorn as part of the Automated Formal
Methods Workshop ([AFM](http://fm.csl.sri.com/AFM17/)) and the Seventh
Summer School on Formal Techniques
([SSSFT](http://fm.csl.sri.com/SSFT17/)) that took place on May 20th
at Menlo College, Atherton (CA). In this post, we reproduce the most
interesting parts of the tutorial.

In the first part of the tutorial, we described the basic usage of
SeaHorn as well as how it can be combined with two interesting
tools: [crab-llvm](github.com/caballa/crab-llvm), an abstract
interpreter for LLVM programs, and [sally](github.com/SRI-CSL/sally),
a model checker for infinite transition systems. In the second part,
we shown a case study on how to prove memory safety using SeaHorn.

# Basic Usage #

We started our tutorial with this simple C program `test.true.c`:

{% highlight c linenos=table %}
#include "seahorn.h"
extern int nd();
int main(void) {  
  int n, k, j;
  n = nd();
  assume (n>0);
  k = nd();
  assume (k>n);
  j = 0;
  while( j < n ) {
    j++;
    k--;
  }
  sassert(k>=0);
  return 0;
}
{% endhighlight %}

The program `test.true.c` is standard C except there are two special
annotations, both defined in the header file `seahorn.h` which is part
of the SeaHorn distribution. The annotation `assume(cond)`, where
`cond` can be any C expression that evaluates to a Boolean, can be
used to either add extra assumptions about the environment or guide
SeaHorn during its search for inductive invariants. The annotation
`sassert(cond)` is equivalent to C `assert`. The reason why we use
`sassert` rather than `assert` is that `sassert` is actually defined
as

{% highlight c %}
 # define sassert(X) if(!(X)) __VERIFIER_error ()
{% endhighlight %}

This definition allows SeaHorn to reduce the validation of an
assertion to the problem of deciding whether the error state denoted
by `__VERIFIER_error` is reachable or not.

Note that there is also in the program an external function `nd`. In
SeaHorn any external function (denoted with C keyword `extern`) that
returns a value of type `ty` is modeled as a function that returns
*non-deterministically* a value of type `ty`. This simple mechanism
allows SeaHorn to have code abstractions. Note that the external
function does not need to be named `nd`. Any external function that
returns an integer will make the trick here.

Let us continue now by running SeaHorn on the program. The first thing
that SeaHorn can do is to translate the C program into LLVM bitcode:

{% highlight c %}
sea fe test.true.c -o test.true.bc
{% endhighlight %}

The option `fe` (front-end) calls `clang` to translate C into bitcode
and performs some preprocessing and optimization of the bitcode. You
can see the content of `test.true.bc` by translating it into
human-readable bitcode format:

{% highlight c %}
llvm-dis test.true.bc -o test.true.ll
{% endhighlight %}

A better option is to use the SeaHorn utility `inspect` as follows:

{% highlight c %}
sea inspect --cfg-dot test.true.bc
dot -Tpng main.dot -o main.dot.png
open main.dot.png // on Linux replace `open` with your favorite `png` viewer
{% endhighlight %}

This command sequence converts first the bitcode to `.dot` format and
then to a `png` file. If you open the `png` file then you should see
something like this:

![Control Flow Graph of test.true.c in LLVM bitcode](images/main.dot.png)

Next, we can use SeaHorn to translate the LLVM bitcode into
verification conditions. We
use [Horn Clauses](https://en.wikipedia.org/wiki/Horn_clause) as the
language to represent verification conditions.

{% highlight c %}
sea horn test.true.bc --step=small -o test.true.smt
{% endhighlight %}

The option `horn` outputs the verification conditions in
`test.true.smt`. By default, this option uses a format that resembles
SMT-lib:

{% highlight lisp %}
;;; declare relations: one per basic block
(declare-rel main@.lr.ph ())
(declare-rel main@_bb (Int Int Int ))
(declare-rel main@orig.main.exit (Int ))
(declare-rel main@orig.main.exit.split ())

;;; declare variables 
(declare-var main@%_8_0 Bool )
(declare-var main@%_9_0 Bool )
(declare-var main@%_10_0 Bool )
(declare-var main@%_11_0 Bool )
(declare-var main@%_7_0 Bool )
(declare-var main@%_1_0 Bool )
(declare-var main@%_3_0 Bool )
(declare-var main@%_0_0 Int )
(declare-var main@%_2_0 Int )
(declare-var main@%j.0.i2_0 Int )
(declare-var main@%k.0.i1_0 Int )
(declare-var main@%_5_0 Int )
(declare-var main@%_6_0 Int )
(declare-var main@%j.0.i2_1 Int )
(declare-var main@%k.0.i1_1 Int )
(declare-var main@%k.0.i1.lcssa_0 Int )

;;; definition of the Horn clauses that have this form
;;;   (rule (=> (and p .... ) p'))
;;; such that there is an edge from p to p' in the CFG of the program

(rule main@.lr.ph) ; entry block is always reachable
(rule (=> (and main@.lr.ph
         true
         (= main@%_1_0 (> main@%_0_0 0))
         main@%_1_0
         (= main@%_3_0 (> main@%_2_0 main@%_0_0))
         main@%_3_0
         (= main@%j.0.i2_0 0)
         (= main@%k.0.i1_0 main@%_2_0))
    (main@_bb main@%j.0.i2_0 main@%k.0.i1_0 main@%_0_0)))
(rule (=> (and (main@_bb main@%j.0.i2_0 main@%k.0.i1_0 main@%_0_0)
         true
         (= main@%_5_0 (+ main@%j.0.i2_0 1))
         (= main@%_6_0 (+ main@%k.0.i1_0 (- 1)))
         (= main@%_7_0 (< main@%_5_0 main@%_0_0))
         main@%_7_0
         (= main@%j.0.i2_1 main@%_5_0)
         (= main@%k.0.i1_1 main@%_6_0))
    (main@_bb main@%j.0.i2_1 main@%k.0.i1_1 main@%_0_0)))
(rule (=> (and (main@_bb main@%j.0.i2_0 main@%k.0.i1_0 main@%_0_0)
         true
         (= main@%_5_0 (+ main@%j.0.i2_0 1))
         (= main@%_6_0 (+ main@%k.0.i1_0 (- 1)))
         (= main@%_7_0 (< main@%_5_0 main@%_0_0))
         (not main@%_7_0)
         (= main@%k.0.i1.lcssa_0 main@%k.0.i1_0))
    (main@orig.main.exit main@%k.0.i1.lcssa_0)))
(rule (=> (and (main@orig.main.exit main@%k.0.i1.lcssa_0)
         true
         (= main@%_8_0 (> main@%k.0.i1.lcssa_0 0))
         (not main@%_9_0)
         (= main@%_10_0 (xor main@%_9_0 true))
         (= main@%_11_0 (= main@%_8_0 false))
         main@%_11_0)
    main@orig.main.exit.split))
	
;;; query	
(query main@orig.main.exit.split)
{% endhighlight %}

TODO: explain here `--step=small` encoding. We can also describe
briefly `--step=large` which is the default one. For the tutorial, we
used `--step=small` because it's simpler but it might be worth it to
present both in detail.

Once we have generated the verification conditions we can use SeaHorn
back-end solvers to try to solve them by adding the option `--solve`:

{% highlight c %}
sea horn --solve test.true.bc --step=small --show-invars
{% endhighlight %}

The default solver used in SeaHorn
is [Spacer](http://bitbucket.org/spacer/code) which is a PDR-based
(PDR refers to Property-Directed Reachability, aka IC3) solver built
on the top of [Z3](https://github.com/Z3Prover/z3). `spacer` infers
inductive invariants over linear integer/real arithmetic and
quantified-free arrays from recursive, non-linear Horn
clauses. `spacer` also computes summaries in presence of
functions. These are the invariants inferred by `spacer`:

{% highlight c %}
unsat

Function: main
main@.lr.ph: true
main@_bb:
		([+
  -1*main@%j.0.i2
  -1*main@%k.0.i1
  main@%_0
]<=-1)
	(main@%k.0.i1>=2)
main@orig.main.exit: (main@%k.0.i1.lcssa>=2)
main@orig.main.exit.split: false
{% endhighlight %}

SeaHorn shows first that the query is `unsat` meaning that the error
location (denoted by `__VERIFIER_error`) cannot be reachable, and
therefore the program is safe!. In addition, SeaHorn outputs for each
basic block the inferred invariants. For instance, for the basic block
`_bb` at `main` (the loop) the invariant is the formula `%j.0.i2 +
%k.0.i1 >= %_0 + 1`. With a bit of faith, you can convince yourself
that `%j.0.i2`, `%k.0.i1`, and `%_0` correspond to the C variables
`j`, `k`, and `n`.

For convenience, SeaHorn allow you to write all the above commands in
a single one:

{% highlight c %}
sea pf test.true.c --step=small --show-invars
{% endhighlight %}

Suppose next a buggy version called `test.false.c`:

{% highlight c linenos=table %}
#include "seahorn.h"
extern int nd();
int main(void) {  
  int n, k, j;
  n = nd();
  k = nd();
  assume(k >= 2);
  j = 0;
  while( j < n ) {
    j++;
    k--;
  }
  sassert(k>=0);
  return 0;
}
{% endhighlight %}

Let us try to prove that it is safe:

{% highlight c %}
sea pf test.false.c --show-invars
{% endhighlight %}

However, the output is different in this case:

{% highlight c %}
sat
main@.lr.ph
main@.lr.ph --> main@.lr.ph
main@.lr.ph --> main@.lr.ph
main@.lr.ph --> main@orig.main.exit.split
main@orig.main.exit.split --> query!1
{% endhighlight %}

Here, the solver outputs `sat` meaning that the error location is
reachable and therefore the program is unsafe. In addition, it outputs
the relations that were reachable during the unsafe trace found by the
solver. Although this can be helpful to understand why the program is
unsafe, it cannot be considered a valid counterexample.

Fortunately, we already saw in one of our previous posts how to
generate automatically executable test harnesses for failed
properties. Let us execute these two commands:

{% highlight c %}
sea pf test.false.c --cex=harness.ll
sea exe -m64 -g test.false.c harness.ll  -o out
{% endhighlight %}

The above commands produce an executable `out` which can be executed
by simply typing:

{% highlight c %}
./out
{% endhighlight %}

You should see a printed message that indicates `__VERIFIER_error` was
reached:

{% highlight c %}
__VERIFIER_error was executed
{% endhighlight %}

This shows that the error location is indeed reachable and therefore
the counterexample found by the solve is an actual counterexample.
More importantly, a key advantage of having an executable is that we
can run now our favorite debugger and inspect why the program failed:

{% highlight c %}
lldb ./out
(lldb) b main
(lldb) run 
   14  	int main(void) {  
   15  	  int n, k, j;
-> 16  	  n = nd();
   17  	  k = nd();
   18  	  assume(k >= 2);
   19  	  j = 0;
(lldb) n
   14  	int main(void) {  
   15  	  int n, k, j;
   16  	  n = nd();
-> 17  	  k = nd();
   18  	  assume(k >= 2);
   19  	  j = 0;
   20  	  while( j < n ) {
(lldb) n
   15  	  int n, k, j;
   16  	  n = nd();
   17  	  k = nd();
-> 18  	  assume(k >= 2);
   19  	  j = 0;
   20  	  while( j < n ) {
   21  	    j++;
(lldb) p n
  (int) $0 = 3  // the value of n
(lldb) p k
  (int) $1 = 2  // the value of k
(lldb) p n      // several times until the exit of the loop is taken

   21  	    j++;
   22  	    k--;
   23  	  }
-> 24  	  sassert(k>=0);
   25  	  return 0;
   26  	}
(lldb) p k
  (int) $2 = -1
(lldb) n
__VERIFIER_error was executed
{% endhighlight %}

The reason for failure was that `n` and `k` can be set initially to
`3` and `2`. Then, the program executes the loop three times and when
the exit of the loop is taken the value of `k` is `-1`.

## SeaHorn and Crab-Llvm ##

[Abstract Interpretation](https://en.wikipedia.org/wiki/Abstract_interpretation) is
a very powerful technique to infer inductive invariants from
transition systems. We show how SeaHorn can be combined
with [Crab-llvm](github/caball/crab-llvm), a tool that infers
invariants from LLVM bitcode based on abstract interpretation, to
strengthen the invariants inferred by `spacer`, the PDR/IC3 back-end
solver. Consider the next example `crab1.c`:

{% highlight c linenos=table %}
#include "seahorn.h"
extern int nd();

int main (){
  int x = 1;
  int y = 0;
  int n = nd();
  assume (n > 0);
  while (nd() && x < n) {
    x=x+y;
    y++;
  }
  sassert (x>=y);
  return 0;
}
{% endhighlight %}

If we run the command:

{% highlight c %}
sea pf -O0 crab1.c --show-invars
{% endhighlight %}

You will notice that SeaHorn is unresponsive for a while. In fact,
SeaHorn will never give you an answer!

To mitigate this, we can tell SeaHorn to call `crab-llvm` and use all
the invariants inferred by `crab-llvm` as *lemmas* in `spacer`:

{% highlight c %}
sea pf -O0 crab1.c --show-invars --crab --crab-dom=int 
{% endhighlight %}

You should see that SeaHorn can now produce an answer. These are the
safe invariants produced by `spacer` together with `crab-llvm`:

{% highlight c %}
unsat
Function: main
main@entry: true
main@_un:
    ((0-main@%y.0.i)<=0)
	main@%_2
	((0-main@%x.0.i)<=-1)
	((main@%y.0.i+(-1*main@%x.0.i))<=0)
main@verifier.error.split: false
{% endhighlight %}

In this example, `spacer` could not find an inductive strengthening of
the property `x >= y`. However, `crab-llvm` can infer using classical
intervals the inductive invariant `y >= 0` which is not enough to
prove the property but when in conjunction with `x >= y` it suffices
to prove the program is safe.

Another example where `spacer` can substantially benefit from abstract
interpretation is when universal quantifiers are needed to prove a
property. Consider the next example `crab2`:

{% highlight c linenos=table %}
#include "seahorn.h"
extern int nd();

#define MAX 100
int main () {
  int n = nd();
  assume (n > 0 && n < MAX);
  int a[MAX];
  int i;
  for (i=0;i<n;i++) {
    a[i]=i;
  }

#ifndef FORALL
  sassert (a[n-1] >= 0);
#else
  int j;
  for (j=0;j<n;j++) {
    sassert (a[j] >= 0);
  }
#endif 
  return 0;
}
{% endhighlight %}

If the following command is executed:

{% highlight c %}
sea pf crab2.c --show-invars
{% endhighlight %}

then, SeaHorn proves the program is safe. Let us try now the command:

{% highlight c %}
sea pf crab2.c -DFORALL --show-invars 
{% endhighlight %}

In this case, we don't get an answer in a reasonable amount of
time. The reason is that we are trying to prove that all elements in
the array are positive. This is tantamount to infer an universally
quantified invariant over all array cells. This is currently beyond
the capabilities of `spacer`. However, `crab-llvm` provides several
array domains that can easily infer invariants like the one we need
for our example. 

We can tell SeaHorn to call `crab-llvm` with an array domain using
intervals to model array contents as follows:

{% highlight c %}
sea pf crab2.c -DFORALL --show-invars  --crab --crab-track=arr --crab-dom=int
{% endhighlight %}

However, note that we don't get an answer yet.

The reason is that even if `crab-llvm` infers that all elements of the
array between `[0,n)` is greater or equal than `0`, `spacer` does not
support lemmas with quantifiers. Therefore, it seems that our effort
of computing universally quantified invariants using `crab-llvm` is
futile. The solution adopted in SeaHorn is to perform explicit
instantiation of the universally quantified invariants inferred by
`crab-llvm` so that `spacer` can digest them as quantifier-free
lemmas. To do that, for each read `v := a[i]` of an array `a` we
insert an assumption `assume (p(v))` where `p` is the property that
holds universally on each array element, and therefore it must hold on
the particular read element `a[i]`. If we execute the command:

{% highlight c %}
sea pf crab2.c -DFORALL --show-invars  --crab --crab-track=arr --crab-dom=int --crab-add-invariants=after-load  --oll=crab2.ll
{% endhighlight %}

we can quickly get the following answer:

{% highlight c %}
unsat
Function: main
main@.lr.ph: true
main@_un:
		((0-main@%i.0.i2)<=0)
	main@%_2
main@precall.split: false
main@postcall:
		((0-main@%j.0.i1)<=0)
	main@%_3
	main@%_2
	main@%_4
{% endhighlight %}


## SeaHorn and Sally ##

In all our previous programs we used `spacer` as the back-end
solver. SeaHorn can also use [sally](github.com/SRI-CSL/sally), a
model-checker for infinite transition systems developed by SRI
folks. Suppose the following example `sally.c`:

{% highlight c linenos=table %}
#include "seahorn.h"
extern int nd();
int main() {
 int x=1; int y=1;
 while(nd()) {
   int t1 = x;
   int t2 = y;
   x = t1+ t2;
   y = t1 + t2;
 }
  sassert(y >=1);
  return 0;
}
{% endhighlight %}

The following command outputs in a file `sally.mcmt` a transition
system in MCMT format which is the format understood by Sally:

{% highlight c %} 
sea smt -O0 sally.c --step=flarge --horn-format=mcmt -o sally.mcmt 
{% endhighlight %}

TODO: add output an excerpt of `sally.mcmt`?

Then, we can run `sally` using its `pdkind` (PDR + k-induction) engine
using the SMT solvers yices2 and MathSat5 (`y2m5`) as follows:

{% highlight c %}
sally --no-input-namespace --engine pdkind sally.mcmt --solver=y2m5 --show-invariant --no-lets
{% endhighlight %}

`sally` returns `valid`.


# Proving Memory Safety with SeaHorn #

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
  size = sz;

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

{% highlight c}
sea pf -O0 abc1_inst.c --show-invars
{% endhighlight %}

SeaHorn can prove the program is safe! (i.e., all array accesses are
in-bounds). The invariants produced by SeaHorn are:

{% highlight c}
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

{% highlight c}
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

TODO: explain the instrumentation

We can now try to prove `abc2_inst.c`:

{% highlight c}
sea pf -O0 abc2_inst.c --show-invars
{% endhighlight %}

And we can also check that the buggy version is indeed unsafe:

{% highlight c}
sea pf -O0 abc2_inst.c --show-invars -DERROR
{% endhighlight %}

TODO: output of these two commands. SeaHorn proves safe `abc2_inst.c`
but it unrolls the program.

Instrument a program so that we can prove memory safety is a lot of
fun! But if you do not want to instrument manually your program,
SeaHorn can add fully automatically the array bounds checks. For
instance, if you execute the command:

{% highlight c}
sea pf -O0 abc2.c --abc=global --show-invars 
{% endhighlight %}

then you should see the following output:

{% highlight c}
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

Well, this is the end of our post where we have described the basics
of SeaHorn as well as how SeaHorn can be combined with other
verification tools. Moreover, we have shown two different ways of
instrumenting a program to prove absence of buffer overflows. Luckily,
SeaHorn can add the instrumentation automatically so that we do not
need to do it by ourselves.






