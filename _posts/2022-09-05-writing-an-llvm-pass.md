---
layout: post
title: Writing an LLVM Pass
date: 2022-09-05 10:00:00
tags: ['LLVM', 'analysis']
---


## Compile LLVM from source

``` shell
git clone git@github.com:llvm/llvm-project.git
pushd llvm-project
    git checkout llvmorg-10.0.1
    mkdir build
    pushd build
        cmake -G "Unix Makefiles" \
            -DLLVM_ENABLE_PROJECTS="clang;libcxx;libcxxabi" \
            -DCMAKE_BUILD_TYPE=DEBUG \
            ../llvm
        cmake --build .
        make -j$(nproc)
    popd
popd
```


## Commands

* Generate LLVM IR: 

```
# Textual form
clang -O1 -emit-llvm input.c -S -o out.ll
# Binary/bit-code form
clang -O1 -emit-llvm input.c -c -o out.bc
```

* Run `HelloWorld` pass with `opt`

``` shell
opt -load-pass-plugin ./libHelloWorld.{so|dylib} -passes=hello-world -disable-output input_for_hello.ll
```