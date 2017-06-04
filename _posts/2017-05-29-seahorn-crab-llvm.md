---
layout: post
title:  "Abstract Interpretation engine in SeaHorn"
subtitle: "SeaHorn and Crab-llvm"
date:   2017-05-29 21:24:45
categories: [seahorn, crab, abstract interpretation]
---

[Abstract Interpretation](https://en.wikipedia.org/wiki/Abstract_interpretation) is
a very powerful technique to infer inductive invariants from
transition systems. We show how SeaHorn can be combined
with [Crab-llvm](https://github.com/caballa/crab-llvm), a tool that infers
invariants from LLVM bitcode based on abstract interpretation, to
strengthen the invariants inferred by `spacer`, the PDR/IC3 back-end
solver. Consider the next example `crab1.c`:

{% highlight c  %}
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

{% highlight c  %}
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

then, SeaHorn proves the program is safe. Let us try now the following command:

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

We can instruct SeaHorn to call `crab-llvm` with an array domain using
intervals to model array contents as follows:

{% highlight c %}
sea pf crab2.c -DFORALL --show-invars  --crab --crab-track=arr --crab-dom=int
{% endhighlight %}

However, note that we don't get an answer yet. The reason is that even if `crab-llvm` infers that all elements of the
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
