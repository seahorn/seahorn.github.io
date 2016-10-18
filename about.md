---
layout: page
title: About SeaHorn
subtitle:
permalink: /about/
---

[SeaHorn](http://seahorn.github.io/) is an automated analysis framework for LLVM-based languages. The key distinguishing feature of [SeaHorn](http://seahorn.github.io/) is its modular
design that separates the concerns of the syntax of the programming
language, its operational semantics, and the verification semantics. SeaHorn
encompasses several novelties:

+ encodes verification conditions using an efficient yet precise inter-procedural technique

+ it provides flexibility in the verification semantics to allow different levels of precision,

+ it leverages the state-of-the-art in software model checking
and abstract interpretation for verification, and

+ it uses Horn-clauses as
an intermediate language to represent verification conditions which simplifies
interfacing with multiple verification tools based on Horn-clauses.

In this blog, we will write different aspects of [SeaHorn](http://seahorn.github.io/).
