---
layout:     post
title:      "Anion & Cation"
date:       2011-08-21 05:54:00
categories:
---
My "official" summer project for Mozilla was to implement a linear scan register allocator for IonMonkey based on the algorithms published by Christian Wimmer. While that was certainly where most of my time went, I had a couple of related side projects that I worked on that require a little bit of background:

Implementing an algorithm from a collection of papers efficiently and correctly is a software engineering challenge. [LinearScan.cpp as of the end of my internship](http://hg.mozilla.org/projects/ionmonkey/file/d826b6da07f2/js/src/ion/LinearScan.cpp) was about 1500 lines long, not to mention about 600 lines of support in [LinearScan.h](http://hg.mozilla.org/projects/ionmonkey/file/d826b6da07f2/js/src/ion/LinearScan.h). Most lines of LinearScan.cpp are algorithmically significant, and keeping track of all of the required invariants can be daunting. To make life more exciting, a violated invariant can lead to a number of symptoms, including:

 - An assertion is tripped somewhere in the allocator
 - An assertion is tripped in a later stage of the compiler, like the code generator
 - No assertions are tripped but the generated code is wrong
 - Correctness remains intact, but the output decreases in some measure of quality 

Of these, tripping an assertion is the allocator itself is obviously preferred: the violated invariant is clear and the assertion probably tripped close to where everything went wrong. Tripping an assertion later means you know what happened, but you might have to dig a little bit to figure out how it got that way. If we generate wrong code (and notice!), it is a lengthy process to identify the erroneous instructions before you can begin to figure out how they got that way. Sub-optimality tends to go unnoticed entirely unless one specifically looks for it.

## The Problem with IonMonkey

In the JavaScript engine proper, Mozilla catches regressions with a staggeringly large regression test suite, [jsfunfuzz](https://bugzilla.mozilla.org/show_bug.cgi?id=jsfunfuzz), [LangFuzz](https://bugzilla.mozilla.org/show_bug.cgi?id=676763), and other automated tools. Performance regressions are caught manually with popular benchmarks or with automated tools such as [Talos](https://wiki.mozilla.org/Buildbot/Talos#) and [AWFY](http://arewefastyet.com/). Unfortunately, IonMonkey is early enough in development that it [can’t compile the majority of the test suite](https://bugzilla.mozilla.org/show_bug.cgi?id=677337), nor can it compile any of the popular benchmarks. jsfunfuzz and LangFuzz test cases are stressful for the runtime, but they aren’t designed to be stressful for a compiler, so these fuzzers are generally not very effective at finding bugs in IonMonkey. To complicate matters, a test that covers edge cases on one revision of the compiler may not on the next: a small change in an early optimization pass can completely change what a later pass sees from the same test.

I recognized some of these issues early in the summer and produced some small ad-hoc python scripts to generate large random test files with lots of nested branches and loops to try to break the bytecode-to-IR construction algorithms. They found about five bugs collectively, but once the issues associated with the code pattern each one produced had been ironed out they were of limited utility.

## Anion

That’s when I started working on Anion, a more general fuzzer tuned to be stressful to IonMonkey. It consists of three parts: a (pluggable) test generator, the test driver, and the test triager. The test generator I used generates a more or less arbitrary JavaScript function using the features that IonMonkey supports, and makes some token attempts to ensure that a reasonable percentage of its output programs can actually run with our current compilation capabilities. The driver generates and runs these scripts, checking their output. If a script causes a crash, it invokes the triager to generate a simplified backtrace ("signature") so that the test case can be organized with others that cause the same breakage. If a script runs to completion or is killed by a timeout, the driver will run the same script in a reference shell and verify that it does the same thing. If the behavior differs, the test case is categorized accordingly.

Anion has automatically found [over thirty bugs](https://bugzilla.mozilla.org/show_bug.cgi?id=674689) in IonMonkey since it was written, and continues to be a valuable bug-finding tool today. It is fast enough to catch some regressions before patches even land and covers a broad enough state space to encounter edge cases we didn’t know were possible.

## Cation

Of course, Anion only finds correctness issues. Identifying performance issues is tricky: we don’t have any good, realistic long-running benchmarks to time yet. However, we can predict performance by counting various occurrences in the compiler such as the emission of a move instruction. Code containing a lot of unnecessary move instructions will likely run more slowly, so if we start emitting a lot of these it probably won’t be good for performance on real code later. To try to catch these performance issues, I wrote Cation.

Cation is a fuzzer similar to Anion, but much simpler. It doesn’t concern itself with crashes or triage, but instead with finding test cases that cause some metric to be worse than some baseline. The output is a pile of test cases that exposed performance issues along with a scorecard of how bad the issue was. Some examples of the sort of things Cation can find test cases for are:

 - Linear scan register allocation producing more move instructions than the greedy allocator
 - Global value numbering increasing register pressure causing the register allocator to emit lots of moves
 - Loop invariant code motion resulting in an increase of the number of instructions in a loop
 - Version A of the compiler producing fewer expensive instructions than later version B on the same function 

Cation makes it very easy to try any of these: simply add print calls in the code wherever the event you want to count occurs. It has already helped me locate several places in the linear scan register allocator where heuristics could be improved, and hopefully it will continue to be helpful for improving performance elsewhere.

## Open Source

Both tools are open source and under active development, and can be checked out from my repository [on GitHub](https://github.com/drakedevel/anion). Hopefully they can be of some use to you, too!
