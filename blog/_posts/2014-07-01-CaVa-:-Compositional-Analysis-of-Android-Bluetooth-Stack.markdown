---
layout: post
title:  "CaVa: Compositional Analysis of Android Bluetooth Software Stack"
---

## Funding Agency ##
NSF

## People ##
* [Temesghen Kahsai][teme] (Principal Investigator) [NASA Ames / CMU]
* [Zvonimir Rakamaric][z] (Principal Investigator) [University of Utah]
* [Marko Dimjašević][marko] (PhD student) [University of Utah]
* [Falk Howar][falk] (External collaborator)

## Description ##
Software protocol stacks are highly heterogeneous and complex, often spanning layers starting from hardware at the bottom, then going through several low-level operating system (OS) layers written in C, and ending up with high-level libraries written in managed languages such as Java. In addition, they are also long-lived entities with lots of legacy code involved. Therefore, developing and maintaining a software protocol stack is a complex task with many parties and products involved over a long period of time. In such scenario there are many issues that impede the developers’ productivity and stack reliability: (1) hardware fragmentation, (2) interaction with other features in a heterogeneous environment, (3) conformance to protocol specification or incomplete specification, (4) problems related to inherent asynchrony and concurrency, (5) real-time timing issues, (6) communication-related problems (overtaking, loss of messages, etc.). It is incredibly hard to tackle all these issues efficiently and correctly for all platforms/devices and all possible states of a system. This naturally leads to many bugs and vulnerabilities in the end product, which is ultimately hindering user experience and even posing security risks to millions of users.

In this project, we plan to develop techniques for the analysis of
such complex systems. We focus on the Android Bluetooth Software
Stack. We plan to develop a compositional analysis technique to increase the reliability of heterogeneous software protocol stacks. Specifically, we will
employ a combination of techniques from automata learning, symbolic
program verification and fault diagnosis. 

[z]: http://www.zvonimir.info/
[falk]: http://www.falkhowar.de/
[teme]: http://www.lememta.info
[marko]: http://dimjasevic.net/marko/
