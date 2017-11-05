---
layout: post
title:  "Bug validation via test harness"
subtitle: "From counter-examples to executable tests"
date:   2016-10-16 14:56:45
categories: [seahorn, cex, validation]
---

Explaining why a piece of code violates a particular specification is one of the most important features that an automated software analysis tool should provide. In fact, tools participating at the International Competition on Software Verification [SV-COMP](http://sv-comp.sosy-lab.org/2017/) are mandated to provide a witness in a [specific format](https://github.com/sosy-lab/sv-witnesses/) that illustrate why a specification is violated. In a second step, the provided witness is validated by external tools (e.g. [CPAChecker](https://cpachecker.sosy-lab.org/) or [Ultimate Automizer](https://monteverdi.informatik.uni-freiburg.de/tomcat/Website/?ui=tool&tool=automizer)). We leave the discussion of the pros and cons of using such fixed format for future blog posts ;).

A few months back, we embarked on a journey to provide an alternative format for witness validation. Our goal was to have a format that is “dynamic” and provide a better understanding of why a property fails. For example, developers routinely use debuggers like [gdb](https://www.gnu.org/software/gdb/) to trace the state of a program at the moment it crashes. So if you want to use a debugger in order to understand why a property is violated, you need a witness that can be executed on a machine.

In this post, we describe a shiny new feature implemented in SeaHorn – the automated generation of executable test harness for failed properties. Instead of providing a fixed format, SeaHorn will provide an object file (the witness) that can be linked with the original program and executed. How does this work in practice? Lets start with a simple piece of C code (`code.c`):

{% highlight c linenos=table %}
#include "seahorn.h"
extern int unknown1(void);

 int main(void) {
 int x=1; int y=1;
 while(unknown1()) {
   int t1 = x;
   int t2 = y;
   x = t1+ t2;
   y = t1 + t2;
  }
   sassert(y < 1);
   return 0;
 }
{% endhighlight %}

 `sassert(...)` is defined in `seahorn.h`

{% highlight c %}
 # define sassert(X) if(!(X)) __VERIFIER_error ()
{% endhighlight %}

In this code, we would like to check if `y<1`. The first step is to run the verification engine of SeaHorn:

{% highlight c %}
sea pf --cex=witness.ll code.c
{% endhighlight %}

If the property is violated, SeaHorn will produce a test harness (`witness.ll`) as [LLVM bitcode](http://llvm.org/docs/BitCodeFormat.html):

{% highlight c %}
; ModuleID = 'harness'
target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"

@0 = private constant [1 x i32] zeroinitializer
@1 = private global i32 0

define i32 @unknown1(...) {
entry:
  %0 = load i32* @1
  %1 = add i32 %0, 1
  store i32 %1, i32* @1
  %2 = call i32 @__seahorn_get_value_i32(i32 %0, i32* getelementptr
                                    inbounds ([1 x i32]* @0, i32 0, i32 0),
                                    i32 1)
  ret i32 %2
}

declare i32 @__seahorn_get_value_i32(i32, i32*, i32)
{% endhighlight %}

In a second step, you can use another option provided in SeaHorn to link the witness object file (`witness.ll`) with the original code (`code.c`):

{% highlight c %}
sea exe -m64 -g code.c witness.ll -o code_debug
{% endhighlight %}

Here, the file `code_debug` is an executable file. In fact, if you run such file:

{% highlight c %}
./code_debug
{% endhighlight %}

it will output:

{% highlight c %}
__seahorn_get_value_i32 0 1
__VERIFIER_error was executed
{% endhighlight %}

Which indicates that `__VERIFIER_error` (the error state) is reachable from the initial state. Hence, the property `y<1` is violated. We are in the process of writing an article to explain in detail the behind the scene of this new feature. Stay tuned :)

The following screencast illustrate a complete run of this new feature:

[![asciicast](https://asciinema.org/a/5jer5yp602ys6x885yl2674yw.png)](https://asciinema.org/a/5jer5yp602ys6x885yl2674yw?autoplay=1")
