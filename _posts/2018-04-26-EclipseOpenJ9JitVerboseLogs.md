---
layout: post
title: Eclipse OpenJ9 Verbose JIT logs
---

This article is the first in a series that provides useful information for analyzing JIT compiler
operations. In this first article I'll explain how to obtain JIT verbose logs and the basics for
how to read them.

A JIT verbose log is a summary of what methods were compiled by the JIT
and some of the compilation heurisic decisions that were taken while the JIT operates
inside the OpenJ9 VM. The basic log provides a lot of interesting information about the
methods being compiled, but there are also some specific suboptions you can add to learn
even more details. In this article, I'll show you how to start getting under the hood
of the Eclipse OpenJ9 JIT compiler..

## OpenJ9 JIT compiler overview

Before we look at verbose logs, I'll quickly cover some high level information about the OpenJ9 JIT and
the JIT compiler optimization levels so that you are familiar with some of the labels in verbose log entries.

When an application runs, methods are initially run by the bytecode interpreter, as they do for virtually all JVMs.
The JIT compiler can improve the performance of an application by compiling the most frequently executing methods
into native machine code at run time.

But the JIT doesn't compile every method that gets called. Instead, OpenJ9 records the number of times a method
is called and when the count reaches a pre-defined *invocation threshold*, JIT compilation is triggered.
Once a method has been compiled by the JIT, the VM can call the compiled method rather than interpreting it.

The JIT compiler can compile a method at different optimization levels: *cold*, *warm*, *hot*,
*very hot with profiling*, or *scorching*. The hotter the optimization level, the better the expected performance,
but the higher the cost in terms of CPU and memory. OpenJ9 implements these different optimization levels
by changing the sequence and types of optimizations that will be performed: higher optimization levels will
perform more optimizations and employ more powerful analyses.

- *cold* is used during startup processing for large applications where the goal is to achieve the compiled code speed for as many methods as possible as quickly as possible.
- *warm* is the workhorse: after start-up, most methods are compiled at warm once they reach the *invocation threshold*.

For higher optimization levels, the VM uses a sampling thread to identify methods that continue to take a lot of time.
Methods that consume more than 1% of the CPU time are compiled at *hot*. Methods that consume more than 12.5% are
scheduled for a *scorching* compilation. However, before that happens the methods are compiled at *very hot with
profiling* and allowed to run for a short time to collect detailed profile data that is then used by the *scorching*
compilation.

When the JIT begins to compile a method, one important analysis it performs, which is important for this article, is method inlining. Inlining brings the bytecodes of methods
called by a method *M* into the body of *M* itself.  When inlined, the JIT compiler can greatly improve the performance of the resulting code. Not only is the overhead of the
call operation itself eliminated but, even more importantly, the JIT compiler can better optimize the operations performed by the inlined methods by taking advantage of the
context under which those methods were called. For example, the compiler may be able to discover that one of the parameters to an inlined method was passed an argument with a
constant value, or that the method is being called inside a loop leading to knowledge about the sequence of arguments that will be passed to the inlined method.

In a verbose log you will see references to the optimization levels chosen for method compilation, but there are
a few other interesting labels that you may see in verbose logs. These other lagels describe additional JIT
technologies and events that may happen because of the JIT compilation heuristics or the behaviour of the
Java application. I'll talk more about these things in later articles, but here are some brief descriptions
just to whet your appetite:

- AOT: Refers to methods that are compiled as "Ahead-Of-Time" code, which is stored into the shared classes cache for subsequent use.
- JNI: Refers to JNI *thunks*, which accelerate calls to a native method.
- DLT: Refers to *Dynamic Loop Transfer* compiled methods, designed to be entered from the loop back edge from an active interpreted method stuck in a long running loop
- OSR: Refers to *On Stack Replacement* (sometimes called "deopt"), where an active stack frame for a JIT-compiled method is replaced with an intepreted stack frame
for the same method. This event may occur, for example, if a breakpoint has been set inside a method that has been JIT compiled.


## Generating a verbose JIT log

Even running a simple command like `java -version` involves JIT compilation. In this example, we will use the `java -version`
command to generate and analyze a verbose log to see some of the methods that the JIT compiles.

It'e really easy to follow along with this article if you have Docker installed on your system. You can start a fully
certified build of OpenJDK8 with Eclipse OpenJ9 distributed by the AdoptOpenJDK community with the following command:

`docker run -it adoptopenjdk/openjdk8-openj9:jdk8u162-b12_openj9-0.8.0 /bin/bash`

When you run that command, you'll be running a basic Ubuntu 16.04 container including an installation of OpenJDK8 with
Eclipse OpenJ9 available on your path as the default `java` command.

To print the verbose log to STDOUT simply add the `-Xjit:verbose` option to the `java` command line. For example:

`java -Xjit:verbose -version`

However, to make it easier to study the log, you can redirect the output to a log file. For example:

`java -Xjit:verbose,vlog=vlogfile -version`

When you run this command the file `vlogfile.<date>.<time>.<JVM_process_ID>` is created in your current directory. Running the command multiple times produces multiple distinct versions of this file.

## Reading the verbose log file

The top section of the log provides information about the environment that the JIT is operating in. You can determine
the version of the JIT and VM that you are using, and the type and number of processors that the JIT has access to.

Underneath you can see the AOT and JIT options that are in force. In the following example output
you can see that the JIT is invoked with the `-Xjit:verbose,vlog=vlogfile` option.

```
#INFO:  _______________________________________
#INFO:  Version Information:
#INFO:       JIT Level  - e24e8aa9
#INFO:       JVM Level  - 20180315_120
#INFO:       GC Level   - e24e8aa9
#INFO:  
#INFO:  Processor Information:
#INFO:       Platform Info:X86 Intel P6
#INFO:       Vendor:GenuineIntel
#INFO:       numProc=1
#INFO:  
#INFO:  _______________________________________
#INFO:  AOT
#INFO:  options specified:
#INFO:       samplingFrequency=2
#INFO:  
#INFO:  options in effect:
#INFO:       verbose=1
#INFO:       vlog=vlogfile
#INFO:       compressedRefs shiftAmount=0
#INFO:       compressedRefs isLowMemHeap=1
#INFO:  _______________________________________
#INFO:  JIT
#INFO:  options specified:
#INFO:       verbose,vlog=vlogfile
#INFO:  
#INFO:  options in effect:
#INFO:       verbose=1
#INFO:       vlog=vlogfile
#INFO:       compressedRefs shiftAmount=0
#INFO:       compressedRefs isLowMemHeap=1
#INFO:  StartTime: Apr 23 09:49:10 2018
#INFO:  Free Physical Memory: 996188 KB
#INFO:  CPU entitlement = 100.00
```

The last few lines of the information section show the start time of the compilation activity, how much free physical memory is available to the process, and the CPU entitlement.


The information section is followed by a sequence of lines that describe the methods that are being compiled as well as other events significant to the operation of the JIT compiler..

Here is a typical line from the verbose log:

```
+ (cold) sun/reflect/Reflection.getCallerClass()Ljava/lang/Class; @ 00007FCACED1303C-00007FCACED13182 OrdinaryMethod - Q_SZ=0 Q_SZI=0 QW=1 j9m=00000000011E7EA8 bcsz=2 JNI compThread=0 CpuLoad=2%(2%avg) JvmCpu=0%
```

In this example:

- The method compiled is `sun/reflect/Reflection.getCallerClass()Ljava/lang/Class`.
- The `+` indicates that this method is successfully compiled. Failed compilations are marked by a `!`.
- `(cold)` tells you the optimization level that was applied. Other examples may be `(warm)` or `(scorching)`.
- `00007FCACED1303C-00007FCACED13182` is the code range where the compiled code was generated.
- `Q` values provide information about the state of the compilation queues when the compilation occurred.
- `bcsz` shows the bytecode size. In this case it is small because this is a native method, so the JIT is simply providing an accelerated JNI transition into the native `getCallerClass` method.

Each line of output represents a method that is compiled.


## Getting more details

If you want to understand more about the performance of the JIT compiler threads you can use the `compilePerformance`
suboption to record additional content into the verbose log. For example:

`java -Xjit:verbose={compilePerformance},vlog=vlogfile -version`

The output generated by using this command adds the values `time` and `mem` into each line, as shown in the following example:

```
+ (cold) java/lang/System.getEncoding(I)Ljava/lang/String; @ 00007F29183A921C-00007F29183A936D OrdinaryMethod - Q_SZ=0 Q_SZI=0 QW=1 j9m=0000000000F13A70 bcsz=3 JNI time=311us mem=[region=704 system=16384]KB compThread=0 CpuLoad=2%(2%avg) JvmCpu=0%
```

- `time=311us` reflects the amount of time taken to do the compilation.
- `mem=[region=704 system=16384]KB` reflects the amount of memory that was allocated during the compilation.

In our example, the method is compiled at the `cold` optimization level, which does not require much memory or CPU.
The higher the optimization level, the more CPU and memory get consumed.

Further suboptions are available to use with the `-Xjit:verbose` command. In this article, we'll look at three more and touch on others in later articles:

- `compileStart` prints out a line when the JIT is about to start compiling a method.
- `compileEnd` prints out a line when the JIT stops compiling a method.
- `inlining` shows the methods that are inlined.

Suboptions can be chained together by using a `|` character. When used, you must enclose the full option name in single quotes (`'`) to avoid the shell misinterpreting these characters as pipe commands.

**Note:** The `compilePerformance` suboption includes the `compileStart` and `compileEnd` suboptions by default, but you can specify these independently of one another. To make it easier to read the output from the `inlining` suboption, you should turn off multiple compilation threads by setting `-XcompilationThreads1` (We plan to improve the output to accommodate multiple compilation threads so that setting this option is not necessary. See [issue 1741](https://github.com/eclipse/openj9/issues/1741)).

The following example can be used to create verbose output that includes lines to show when compilation for a method starts and ends, and any methods that are inlined during the compilation.

`java '-Xjit:verbose={compileStart|compileEnd|inlining},count=5,vlog=vlogfile' -XcompilationThreads1 -version`

To make it easier to observe more substantial JIT compilations while just running `java -version`, we will use `count=5`, which causes Java methods to be compiled after just 5 invocations. Using this option can make performance dramatically worse, so I don't recommend it for general use. I'm just adding this option here so we can get more interesting inlining output even while running the decidedly uninteresting `java -version` command.

The following section is taken from the output and describes the compilation and inlining of one method `java/lang/String.equals`:

```
(warm) Compiling java/lang/String.equals(Ljava/lang/Object;)Z  OrdinaryMethod j9m=0000000001300B30 t=90 compThread=0 memLimit=262144 KB freePhysicalMemory=969 MB
#INL: 7 methods inlined into 4dce72bd java/lang/String.equals(Ljava/lang/Object;)Z @ 00007F53190A3E40
#INL: #0: 4dce72bd #-1 inlined 4dce72bd@22 -> 81670d20 bcsz=37 java/lang/String.lengthInternal()I
#INL: #1: 4dce72bd #-1 inlined 4dce72bd@28 -> 81670d20 bcsz=37 java/lang/String.lengthInternal()I
#INL: #2: 4dce72bd #-1 inlined 4dce72bd@104 -> bf62dcaf bcsz=182 java/lang/String.regionMatchesInternal(Ljava/lang/String;Ljava/lang/String;[C[CIII)Z
#INL: #3: 4dce72bd #2 inlined bf62dcaf@121 -> bbb5af92 bcsz=39 java/lang/String.charAtInternal(I[C)C
#INL: #4: 4dce72bd #2 inlined bf62dcaf@131 -> bbb5af92 bcsz=39 java/lang/String.charAtInternal(I[C)C
#INL: #5: 4dce72bd #2 inlined bf62dcaf@156 -> bbb5af92 bcsz=39 java/lang/String.charAtInternal(I[C)C
#INL: #6: 4dce72bd #2 inlined bf62dcaf@166 -> bbb5af92 bcsz=39 java/lang/String.charAtInternal(I[C)C
#INL: 4dce72bd called 4dce72bd@120 -> f734b49c bcsz=233 java/lang/String.deduplicateStrings(Ljava/lang/String;Ljava/lang/String;)V
#INL: 4dce72bd coldCalled 4dce72bd@104 -> bf62dcaf bcsz=182 java/lang/String.regionMatchesInternal(Ljava/lang/String;Ljava/lang/String;[C[CIII)Z
#INL: 4dce72bd coldCalled 4dce72bd@104 -> bf62dcaf bcsz=182 java/lang/String.regionMatchesInternal(Ljava/lang/String;Ljava/lang/String;[C[CIII)Z
+ (warm) java/lang/String.equals(Ljava/lang/Object;)Z @ 00007F53190A3E40-00007F53190A40D0 OrdinaryMethod - Q_SZ=277 Q_SZI=277 QW=1667 j9m=0000000001300B30 bcsz=127 GCR compThread=0 CpuLoad=2%(2%avg) JvmCpu=0%
```


The first line is included as a result of setting the `compileStart` suboption and shows the start of the `warm` method compilation:

`(warm) Compiling java/lang/String.equals(Ljava/lang/Object;)Z  OrdinaryMethod j9m=0000000001300B30 t=90 compThread=0 memLimit=262144 KB freePhysicalMemory=969 MB`

Similarly, the last line shows the successful compilation of this method, as denoted by the `+`:

`+ (warm) java/lang/String.equals(Ljava/lang/Object;)Z @ 00007F53190A3E40-00007F53190A40D0 OrdinaryMethod - Q_SZ=277 Q_SZI=277 QW=1667 j9m=0000000001300B30 bcsz=127 GCR compThread=0 CpuLoad=2%(2%avg) JvmCpu=0%`

The lines inbetween that start with `#INL` describe the inlining operations that took place. A total of 7 methods were inlined into `java/lang/String.equals`:

- The first three methods (`#0, #1, #2`) are inlined into the top level method, denoted as `#-1`:

    ```
    #INL: #0: 4dce72bd #-1 inlined 4dce72bd@22 -> 81670d20 bcsz=37 java/lang/String.lengthInternal()I
    #INL: #1: 4dce72bd #-1 inlined 4dce72bd@28 -> 81670d20 bcsz=37 java/lang/String.lengthInternal()I
    #INL: #2: 4dce72bd #-1 inlined 4dce72bd@104 -> bf62dcaf bcsz=182 java/lang/String.regionMatchesInternal(Ljava/lang/String;Ljava/lang/String;[C[CIII)Z
    ```

- The next four methods (`#3, #4, #5, #6`) are inlined into the method denoted by `#2`.

    ```
    #INL: #3: 4dce72bd #2 inlined bf62dcaf@121 -> bbb5af92 bcsz=39 java/lang/String.charAtInternal(I[C)C
    #INL: #4: 4dce72bd #2 inlined bf62dcaf@131 -> bbb5af92 bcsz=39 java/lang/String.charAtInternal(I[C)C
    #INL: #5: 4dce72bd #2 inlined bf62dcaf@156 -> bbb5af92 bcsz=39 java/lang/String.charAtInternal(I[C)C
    #INL: #6: 4dce72bd #2 inlined bf62dcaf@166 -> bbb5af92 bcsz=39 java/lang/String.charAtInternal(I[C)C
    ```

Here's how to interpret the line for `#INL: #0`:

- The method is inlined into `4dce72bd`, where `4dce72bd` is an internal pointer that corresponds to this method (in this case, `java/lang/String.equals(Ljava/lang/Object;)Z`).
- The value `@22` at the end of the pointer is a bytecode index, which describes the bytecode index of the call that is being inlined.
- The call is `81670d20 bcsz=37 java/lang/String.lengthInternal()I`, which shows the corresponding internal pointer, bytecode size (bcsz) and the name of the method that got inlined.

Going through the `#INL` output line by line then:

- `java/lang/String.lengthInternal()I` got inlined into its caller `4dce72bd` at bytecode index `@22`.
- `java/lang/String.lengthInternal()I` also got inlined into its caller `4dce72bd` at bytecode index `@28`.
- `java/lang/String.regionMatchesInternal(...)` got inlined at call reference `4dce72bd` at bytecode index `@104`.

Then 4 distinct calls to `java/lang/String.charAtInternal(I[C)C` were also inlined into `java/lang/String.regionMatchesInternal(...)` :
- `#3` at bytecode index `@121` of `regionMatchesInternal`
- `#4` at bytecode index `@131` of `regionMatchesInternal`
- `#5` at bytecode index `@156` of `regionMatchesInternal`
- `#6` at bytecode index `@166` of `regionMatchesInternal`

These were all the calls that the inliner decided to inline into the method being compiled. There is some additional output that describes
calls to methods that weren't inlined:

```
#INL: 4dce72bd called 4dce72bd@120 -> f734b49c bcsz=233 java/lang/String.deduplicateStrings(Ljava/lang/String;Ljava/lang/String;)V
#INL: 4dce72bd coldCalled 4dce72bd@104 -> bf62dcaf bcsz=182 java/lang/String.regionMatchesInternal(Ljava/lang/String;Ljava/lang/String;[C[CIII)Z
#INL: 4dce72bd coldCalled 4dce72bd@104 -> bf62dcaf bcsz=182 java/lang/String.regionMatchesInternal(Ljava/lang/String;Ljava/lang/String;[C[CIII)Z
```

While the output does not specifically say why these methods were not inlined, the relatively larger bytecode size (`bcsz=233`) probably prevented the first
method from being inlined. It's possible that, at a higher optimization level than `cold`, this `deduplicateStrings` method may get inlined.
The `coldCalled` label on the last two lines, however, indicate that these calls are located in a part of the method that has not ever been executed, so
the JIT decided that inlining those last two methods will probably increase compile time without much promise that it will improve performance.


By reading the log in this way you can reconstruct the tree of inlines that are taking place as the compilation proceeds.
You can see which methods are being inlined and which methods are not being inlined.

At this point, you should be able to collect a verbose JIT log for your application and better understand what the JIT compiler is doing as it
compiles the code in your application. I hope you'll give it a try!  If you have any questions or suggestions to improve the logs, please
visit out GitHub project and open an [issue](https://github.com/eclipse/openj9/issues) or join our
[community call](https://mstoodle.github.io/EclipseOpenJ9CommunityCall/) and tell us about your experience!

I recently did a live demo covering the information in this article as part of the weekly Eclipse OpenJ9 community call.
You can check it out on Youtube if you'd like (about 14 minutes):
[Eclipse OpenJ9: verbose JIT "lightning" talk](https://youtu.be/v2Q7joRoH9g ).

*written by:*

*Mark Stoodley, an Eclipse OpenJ9 project lead (Twitter: @mstoodle)*
with grateful credit to *Sue Chaplain* for transcribing the demo and polishing the text.
