---
layout: post
title:  "CoCo: Safety Analysis of Flight Critical Systems"
---

## Funding Agency ##
NASA : System-Wide Safety Assurance Technologies Project

## People ##
* [Temesghen Kahsai][teme] (Principal Investigator) [NASA Ames / CMU]
* [Arie Gurfinkel][arie] (Co-Investigator) [SEI / CMU]
* [Cesare Tinelli][cesare] (Co-Investigator) [University of Iowa]
* [Falk Howar][falk] (External Collaborator) 

## Description ##
A common practice in the development of complex component-based Flight Critical Systems (FCS) is to outsource the implementation of some of the components to external contractors or assemble them from Commercial Off-The-Shelf (COTS) systems. Those components are delivered as black-box systems, many of which are first prototyped in- house by the system integrator. Traditionally in this sort of development scenario, the verification of the assembled entire system is done using black-box testing. Despite its limitation, the latter has long been a preferred verification technique.

In this project we seek to achieve a higher level of safety assurance than traditional black-box testing. Specifically, the project will develop strategies for formal verification techniques that take advantage of design models and early prototype component implementations to achieve this higher degree of assurance once the entire system is assembled from outsourced black-box components. The proposed approach will be based on techniques developed in the area of model inference, invariant generation, model checking and automated assume/guarantee reasoning. The project focuses on systems initially specified by means of Simulink/StateFlow models, the de-facto modeling standard in avionics industry, and on C/C++ prototypical implementations of in-house developed components.




[arie]: http://arieg.bitbucket.org/
[cesare]: http://homepage.cs.uiowa.edu/~tinelli/
[falk]: http://www.falkhowar.de/
[teme]: http://www.lememta.info
