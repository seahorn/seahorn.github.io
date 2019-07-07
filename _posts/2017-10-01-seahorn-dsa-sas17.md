---
layout: post
title:  "A Context-Sensitive Memory Model for Verification of C/C++ Programs"
subtitle: "New paper in SAS'17"
date:   2017-10-01 15:56:45
categories: [seahorn, memory]
---

We have recently published a paper in the 24th Static Analysis
Symposium
([SAS'17](http://staticanalysis.org/sas2017/sas2017.html)). The paper
titled
[A Context-Sensitive Memory Model for Verification of C/C++ Programs](http://seahorn.github.io/papers/sea-dsa-SAS17.pdf) describes
a context-, field-, and array-sensitive pointer analysis for C and C++
programs. The slides used in the presentation can be
found [here](http://seahorn.github.io/papers/sas17_slides.pdf).

The pointer analysis produces a finer-grained partition of memory so
that SeaHorn can generate verification conditions involving complex
low-level pointers and arrays constraints but yet they can be solved
efficiently. 

The code of the analysis is available
in [sea-dsa](https://github.com/seahorn/sea-dsa). sea-dsa is now the
default pointer analysis used in SeaHorn, replacing the
original [llvm-dsa](https://github.com/seahorn/llvm-dsa).  Note that
sea-dsa is a stand-alone project so you can use it in your analysis
tool!


