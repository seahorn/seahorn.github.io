---
layout: post
title:  "Using SeaHorn"
subtitle: "Simple safety anaysis"
date:   2016-10-15 14:56:45
categories: [seahorn, usage, safety]
---

How to use SeaHorn?

Given a simple C code (`code.c`)

{% highlight C linenos %}
#include "seahorn/seahorn.h"
int unknown1();

int main()
{
 int x=1; int y=1;
 while(unknown1()) {
   int t1 = x;
   int t2 = y;
   x = t1+ t2;
   y = t1 + t2;
 }
  sassert(y >=1);
}
{% endhighlight %}

you can use SeaHorn verification engine as follows:

{% highlight C linenos %}
sea pf --horn-stats code.c
{% endhighlight %}
