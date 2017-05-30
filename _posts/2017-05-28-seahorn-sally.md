---
layout: post
title:  "A different backend solver for SeaHorn"
subtitle: "Sally as a backend solver for SeaHorn"
date:   2017-05-28 21:24:45
categories: [seahorn, sally]
---

In all our previous programs we used `spacer` as the back-end
solver. SeaHorn can also use [sally](https://github.com/SRI-CSL/sally), a
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
