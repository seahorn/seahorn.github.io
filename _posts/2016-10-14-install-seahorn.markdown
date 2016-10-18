---
layout: post
title:  "Installing SeaHorn"
subtitle: "Feel home!"
date:   2016-10-14 23:34:01
categories: [seahorn, install]
---


* `cd seahorn ; mkdir build ; cd build`
* `cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=run ../ `
* (optional) `cmake --build . --target extra` to download extra packages
* `cmake --build .` to build dependencies (Z3 and LLVM)
* (optional) `cmake --build .` to build extra packages (crab-llvm)
* `cmake --build .` to build seahorn
* `cmake --build . --target install` to install everything in `run` directory
