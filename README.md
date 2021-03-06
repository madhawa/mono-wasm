# mono-wasm

This project is a proof-of-concept aiming at building C# applications into WebAssembly, by using Mono and  compiling/linking everything statically into one .wasm file that can be easily delivered to browsers.

The process does not use Emscripten but instead uses the experimental WebAssembly backend of LLVM, the LLVM linker and the binaryen tooling to generate the final .wasm code.

The final .wasm file is loaded from JavaScript (see `index.js`), which also exposes proper callbacks for system calls that the C library will be calling into. These syscalls are responsible for heap management, I/O, etc.

This project is a work in progress. Feel free to ping me if you have questions or feedback: laurent.sansonetti@microsoft.com

## Related repositories

* [mono-wasm-mono](https://github.com/lrz/mono-wasm-mono): a fork of Mono with changes for a wasm32 target, used to build the runtime and compiler.
* [mono-wasm-libc](https://github.com/lrz/mono-wasm-libc): a fork of the WebAssembly/musl C library with tweaks for our version of Mono and our JS glue.

## Current status

This is a work in progress, but you can see `hello.cs` running here: www.hipbyte.com/~lrz/mono-wasm-hello

## How does it work?

An ASCII graph is worth a thousand words:

```
+----------------+-------------+  +---------------------+
|  Mono runtime  |  C library  |  |    C# assemblies    | <-------+
+----------------+-------------+  +----------+----------+         |
           clang |                           | mono               |
  -target=wasm32 |                           | -aot=llvmonly      |
                 v                           v                    |
+-------------------------------------------------------+         |
|                       LLVM bitcode                    |         |
+----------------------------+--------------------------+         |
                             | llc                                |
                             | -march=wasm32                      | load
                             v                                    | metadata
+-------------------------------------------------------+         | (runtime)
|                   LLVM WASM assembly                  |         |
+----------------------------+--------------------------+         |
                             | s2wasm                             |
                             | + wasm-as                          |
                             v                                    |
+-------------------------------------------------------+         |
|                        index.wasm                     |---------+
+----------------------------------------+--------------+               
                 ^                       | libc                         
   load, compile |                       | syscalls                     
    + run main() |                       v                             
+----------------+--------------------------------------+         +-----------+ 
|                         index.js                      | <-----> |  Browser  |
+-------------------------------------------------------+         +-----------+
```

## Build instructions

We will assume that you want to build everything in the ~/src/mono-wasm directory.

```
$ mkdir ~/src/mono-wasm
$ cd ~/src/mono-wasm
$ git clone git@github.com:lrz/mono-wasm.git build
```

### LLVM+clang with WebAssembly target

We need a copy of the LLVM tooling (clang included) with the experimental WebAssembly target enabled. Make sure to build a Release build (as indicated below) otherwise the WASM codegen will be significantly slower.

```
$ cd ~/src/mono-wasm
$ svn co http://llvm.org/svn/llvm-project/llvm/trunk llvm
$ cd llvm/tools
$ svn co http://llvm.org/svn/llvm-project/cfe/trunk clang
$ cd ../..
$ mkdir llvm-build
$ cd llvm-build
$ cmake -G "Unix Makefiles" -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD=WebAssembly -DCMAKE_BUILD_TYPE=Release ../llvm
$ make
```

After you did this you should have the `~/src/mono-wasm/llvm-build/bin/llc` and `~/src/mono-wasm/llvm-build/bin/clang` programs built with the wasm32 target.

```
$ ~/src/mono-wasm/llvm-build/bin/llc --version
LLVM (http://llvm.org/):
  LLVM version 5.0.0svn
  DEBUG build with assertions.
  Default target: x86_64-apple-darwin15.6.0
  Host CPU: haswell

  Registered Targets:
[...]
    wasm32     - WebAssembly 32-bit
    wasm64     - WebAssembly 64-bit
```

```
~/src/llvm-build/bin/clang --version
clang version 5.0.0 (trunk 306818)
Target: x86_64-apple-darwin15.6.0
Thread model: posix
InstalledDir: /Users/lrz/src/mono-wasm/llvm-build/bin

  Registered Targets:
[...]
    wasm32     - WebAssembly 32-bit
    wasm64     - WebAssembly 64-bit
```

### binaryen tools

We need to build the binaryen tools, that we will use to convert the "assembly" generated by LLVM into WebAssembly text then code.

```
$ cd ~/src/mono-wasm
$ git clone git@github.com:WebAssembly/binaryen.git
$ cd binaryen
$ cmake .
$ make
```

After you did this you should have a bunch of executables available. We will need `s2wasm` and `s2wasm`.

```
$ ls ~/src/mono-wasm/binaryen/bin
asm2wasm        s2wasm          wasm-dis        wasm-shell
binaryen.js     wasm-as         wasm-merge      wasm.js
empty.txt       wasm-ctor-eval  wasm-opt        
```

### Mono compiler

We need a build a copy of the Mono compiler that we will use to generate LLVM bitcode from assemblies. We are building this for 32-bit Intel (i386) because the Mono compiler assumes way too many things from the host environment when generating the bitcode, so we want to match the target architecture (which is also 32-bit).

First, you need to build a copy of the Mono fork of LLVM. We are building it for both 32-bit and 64-bit Intel so that we can easily switch the Mono compiler back to 64-bit later, and we manually have to copy the headers to the build directory as the Mono build system doesn't support external LLVM builds.

```
$ cd ~/src/mono-wasm
$ git clone git@github.com:mono/llvm.git llvm-mono
$ mkdir llvm-mono-build
$ cd llvm-mono-build
$ cmake -G "Unix Makefiles" -DCMAKE_OSX_ARCHITECTURES="i386;x86_64" ../llvm-mono
$ ditto ../llvm-mono/include include
$ make
```

Now, we can now build the Mono compiler itself.

```
$ cd ~/src/mono-wasm
$ git clone git@github.com:lrz/mono-wasm-mono.git mono-compiler
$ cd mono-compiler
$ ./autogen.sh --host=i386-darwin --with-cross-offsets=offsets-wasm32.h CFLAGS="-DCOMPILE_WASM32 -DMONO_CROSS_COMPILE" CXXFLAGS="-DCOMPILE_WASM32 -DMONO_CROSS_COMPILE" --disable-boehm --with-sigaltstack=no --enable-llvm --enable-llvm-runtime --with-llvm=../llvm-mono-build --disable-btls --with-runtime_preset=testing_aot_full
$ make
```

At the end of this process you should have a `mono` executable installed as `~/src/mono-wasm/mono-compiler/mono/mini/mono` built for the i386 architecture.

```
$ file mono/mini/mono
mono/mini/mono: Mach-O executable i386
```

### Mono runtime

Now we can prepare the Mono runtime. We have to clone a new copy of the source code. We are not building the runtime code using the Mono autotools system, so we have to copy header files that are normally generated.

```
$ cd ~/src/mono-wasm
$ git clone git@github.com:lrz/mono-wasm-mono.git mono-runtime
$ cd mono-runtime
$ cp config-wasm32.h config.h
$ cp eglib/src/eglib-config-wasm32.h eglib/src/eglib-config.h
```

### C library

Similarly as above, we clone a copy of the C library that we will be using.

```
$ cd ~/src/mono-wasm
$ git clone git@github.com:lrz/mono-wasm-libc.git libc
```

### OK ready!

We are ready to build our Hello World.

```
$ cd ~/src/mono-wasm/build
$ vi Makefile               # make sure the *_PATH variables point to proper locations, should be the case if you followed these instructions
$ make
```

This will build the mono runtime as LLVM bitcode, then the libc as LLVM bitcode, then our assemblies as LLVM bitcode (using the compiler built above), link everything together, generate an `index.wasm` file, and run it with the `d8` tool.

## TODO

TODO (now):

* do a C# IR linker pass to reduce the size of the assemblies (mscorlib is currently too big)
* work on patches for mono based on the changes made in the fork
* merge the WebAssembly LLVM code into the mono/llvm fork so that the compiler can target wasm32
* write a simple tool that does the IR -> wasm generation (instead of calling llc + the binaryen tools) so that we can better control linking optimizations (ex. LTO)
* investigate threads, sockets, debugger, stack unwinding (+ scanning for the GC), simd and atomic operations, etc.
