---
layout: post
title:  "New paper in PLDI'19"
subtitle: "Static Analysis of Linux Kernel Extensions (aka eBPF programs)"
date:   2019-07-04 10:26:05
categories: [seahorn, crab, static analysis, linux extensions, eBPF]
---

We have recently presented our paper "Simple and Precise Static
Analysis of Untrusted Linux Kernel Extensions" in PLDI 2019. The paper
has been a joint work with our colleagues at VMware Research and Tel
Aviv University. The
[paper](https://jorgenavas.github.io/papers/ebpf-pldi19.pdf) is about
static analysis of eBPF programs using our Crab abstract
interpretation framework.

But, what are eBPF programs?

[eBPF](https://lwn.net/Articles/740157/) allows running user space
programs inside the kernel. It was originally created to speed up
the filtering of network packets. Before kernel extensions, packets
needed to be copied from the kernel to the user space so that they
could be filtered there. With kernel extensions, packets can be
directly filtered inside the kernel avoiding unnecessary copies to the
user space. Originally, it was simply called BPF (Berkeley Packet
Filtering). In 2014, Alexei Starovoitov from Facebook redesigned the
language (eBPF refers to extended BPF) so that very efficient
executable code can be generated taking advantage of modern
hardware. eBPF programs are Turing-complete and today, they have many
interesting uses beyond packet filtering such as profiling, kernel
debugging, security policies, etc.

![eBPF architecture](http://seahorn.github.io/images/ebpf-arch.jpg "eBPF architecture")

This sounds great, but why are we interested in eBPF programs?

Since eBPF programs are executed in the kernel, it is mandatory to
ensure that the program cannot crash, no information can be leaked
from kernel to user space, and it always terminates. To do that, the
Linux kernel has its own verifier to guarantee that. Yes, there is a
verifier as part of the kernel. If you don't believe me
check
[this](https://elixir.bootlin.com/linux/latest/source/kernel/bpf/verifier.c) out.

The verifier walks eBPF programs in depth-first search while checking
all paths for invalid memory accesses. Termination is trivially
ensured by not allowing loops. The number of paths grows exponentially
with the number of branches. Thus, if the number of paths exceeds a
certain threshold the program is rejected. In summary, the verifier
has several important limitations that we try to tackle in the paper:

- Too many false positives: many correct programs are rejected
- Lack of scalability: cannot handle programs with many paths
- Does not support loops
- Lack of formal foundations

All these limitations are well known by eBPF developers. As an
example, we quote here Jakub Sitnicki from CloudFlare:

```
"If you've spent any time using eBPF, you must have experienced first hand the
dreaded eBPF verifier. It's a merciless judge of all eBPF code that will reject any
programs that it deems not worthy of running in kernel-space."
```

To mitigate all these problems, we propose a new solution based
on
[abstract interpretation](https://en.wikipedia.org/wiki/Abstract_interpretation) and
develop a new memory abstract domain on the top of our Crab
framework. Our evaluation with 192 programs from real-world projects
such as Linux, Cilium, OpenVSwitch, and Suricata shows that we can
verify the programs very fast (each one in less than 5 seconds) and
with a very low false positive ratio (we proved all programs except
one). Even more importantly, our verifier accepts programs with loops
that were not possible to write before.

That's all for this post. 

Please, read
our [paper](https://jorgenavas.github.io/papers/ebpf-pldi19.pdf) to
learn more about how we formalize eBPF programs, how we verify them,
and more about our evaluation. Our new verifier is called PREVAIL and
is publicly available [here](https://vbpf.github.io/).

![verification as enabler](http://seahorn.github.io/images/starovoitov-tweet.png)



