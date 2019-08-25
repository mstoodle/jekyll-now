---
layout: post
title: Building OMR JitBuilder with CMake
---

The JitBuilder library in the Eclipse OMR project is designed to make it easier than ever
for people to write JIT compilers, leveraging the high performance JIT compiler component
of OMR, which is used at the heart of the Eclipse OpenJ9 Java Virtual Machine and the
IBM SDK for Java.

In my previous articles about JitBuilder, I've introduced the library and described how
to build it from source.  The Eclipse OMR project, however, has since gone through a long,
at times tortuous, process to migrate its build configuration management from an autotools
based approach to use CMake[].  This article is not about the migration, but rather about
how that migration impacted the steps to build thje JitBuilder library and find the build
artifacts it produces.

I'm going to start from the very beginning and assume you don't even have the Eclipse OMR
project on your system (what are you waiting for?!?). I'll give the directions required
when I build it on MacOS but, as you'll see, there isn't anything MacOS-specific about the
directions so they should really work anywhere the OMR project can run!

The first step is simply to clone the Eclipse OMR repository from GitHub:

```sh
$ git clone https://github.com/eclipse/omr
```

Once you have the project cloned, the next step is to set up the CMake configuration
process, which generates the Makefiles that will be used to actually build the project
and the JitBuilder library. It's actually really easy to prepare to build with CMake:

```sh
$ cd omr
$ mkdir build
$ cd build
```

One of the great improvements we get with CMake is that the builds are all "out of tree"
meaning that the build no longer drops object files into the directories where the source
code is stored. This `build` directory you just created is the root for where those object
files and libraries are going to be generated. You can call it anything you want, and you
can even create different build directories that represent different kinds of builds (e.g.
Release versus Debug).

The next step is to run `cmake`, which will do all the platform detection and configuration
steps and generate the required makefiles:

```sh
$ cmake .. -DOMR_COMPILER=1 -DOMR_TEST_COMPILER=1 -DOMR_JITBUILDER=1 -DCMAKE_BUILD_TYPE=Release
```

The first three -D options ensure that the OMR compiler component, the compiler tests and
the JitBuilder library will be built. The last one tells cmake to build libraries with
optimization, which is particularly important for the OMR compiler and hence the JitBuilder
library so that JitBuilder compile times are as good as they can be.

With that configuration completed, all that's left is to build the project:

```sh
$ make -j<N>
```

When this command completes, you will have successfully built the JitBuilder library!

There are three main artifacts related to the JitBuilder library that get built:
1. JitBuilder static library
2. include files that declare the JitBuilder API
3. code samples that demonstrate how to use various bits of the API

Unfortunately, these three bits aren't all collected together (though hopefully I'll
be updating this article soon after we've made it all simpler!). Here's the current state
of things...

The JitBuilder static library itself can be found here: `omr/build/jitbuilder/libjitbuilder.a`
This library is the one you would link against your own code that makes calls into the
JitBuilder API.

How would your program know what calls it can makes? There are header files for that!
The JitBuilder header files can be found here: `omr/jitbuilder/release/cpp/include` .
Why there? Well, historically and before we adopted CMake, the JitBuilder library would
build itself into that `release` directory. We'll be moving those files into a more
consistent location with the library. The reason these files are in a `cpp` directory is
that the we are working to make the JitBuilder library available in multiple languages,
so these include files are the ones that you would use from C++. There is a C API that
has been created as well, and a Python JitBuilder API is in the works among others!

Want to see some standalone examples for how the JitBuilder API can be used? We have many
code samples available in the `omr/jitbuilder/release/cpp/samples` directory that show
how you can use JitBuilder from C++. The simplest one is called `Simple.cpp` which
creates a function at run time that just returns the integer 3. One of the more complicated
examples dynamically compiles and runs the Mandelbrot program. It's not that I think
people would want to compile Mandelbrot dynmamically rather than statically or that it
confers some advantage to do it dynamically: these are just code samples to show how the API
works and how it can be used to compile not just trivial code examples.

In  the next article, I'm going to cover the major parts of the JitBuilder API and explain
how they are typically used.
