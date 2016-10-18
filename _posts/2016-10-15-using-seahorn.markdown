---
layout: post
title:  "Using SeaHorn"
subtitle: "Simple safety analysis"
date:   2016-10-15 14:56:45
categories: [seahorn, usage, safety]
---

How to use SeaHorn? Given a simple C code (`code.c`)

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

{% highlight C %}
sea pf --horn-stats code.c
{% endhighlight %}

this will output a bunch stuff including the following:

{% highlight C %}
************** BRUNCH STATS *****************
BRUNCH_STAT Result TRUE
BRUNCH_STAT Horn 0.01
BRUNCH_STAT HornClauseDB::loadZFixedPoint 0.00
BRUNCH_STAT HornifyModule 0.00
BRUNCH_STAT LargeHornifyFunction 0.00
BRUNCH_STAT seahorn_total 0.01
************** BRUNCH STATS END *****************
{% endhighlight %}
