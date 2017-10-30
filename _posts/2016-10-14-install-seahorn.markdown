---
layout: post
title:  "Installing SeaHorn"
subtitle: "Feel home!"
date:   2016-10-14 23:34:01
categories: [seahorn, install]
---

Currently, the best way to install SeaHorn is to build it from the source code.

* `cd seahorn ; mkdir build ; cd build`
* `cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=run ../ `
* (optional) `cmake --build . --target extra` to download extra packages
* `cmake --build .` to build dependencies (Z3 and LLVM)
* (optional) `cmake --build .` to build extra packages (crab-llvm)
* `cmake --build .` to build seahorn
* `cmake --build . --target install` to install everything in `run` directory

A common problem after installation is a mismatch between Clang and
LLVM versions. SeaHorn installs LLVM 3.8 so make sure you SeaHorn can
find Clang 3.8.
