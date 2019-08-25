---
layout: post
title: In the Beginning, There Were Programs
---

# Order

In the beginning, there were programs. Programs generally consisted of statements which ran in order. The order was originally an actual physical ordering: instructions on a sequential tape, or a sequence of punch cards. Order later became order of lines or "statements" in a text file. Some code may be conditionally executed, and we could jump around with `GOTO`'s, loops, subroutines, functions, etc. But programs basically consisted of chunks of statements executed in order.

In the beginning, all code executed in a local environment. The hardware hid nasty details from us, such as the delay between issuing a memory-write command on the CPU and that value being available for read. Our statements executed on single thread, and we didn't worry much about "effects" and potential synchronization issues, because everything was made to appear synchronous.

In the beginning, input was static. We had a batch of data that went in to our program, and later we'd get some batch of output. The program was ephermeral, lasting only long enough to either process the input or perhaps fail. Programming languages perhaps were not "functional" as defined today, but the program itself (often) exhibited mathematical purity.

# Chaos

Fast forward to today. Batch programs still exist, but much of the effort in software deals with distributed "always on" systems: 
* mobile/web clients (often themselves containing significant business logic), 
* microservices, 
* various persistent stores (databases, KV stores, immutable logs, message queues, etc.), 
* third-party services,
* and so on.

For such applications, we no longer have the nice clean situation of a single batch of input, transformed by sequential statements to produce a unique batch of output. There are multiple points of input (client, message queues, responses to requests from DBs or other services), these change over time with the state of the system, with no guarantees as to which order the occur. We can try to force such guarantees, but in real distributed system this almost always leads to undesirable effects such as poor performance.

In the beginning, we could rely on the local and sequential nature of code execution. Many _logical_ constraints of the system were enforced simply as _temporal_ constraints: we can assume _X_ is true at line 50 because we asserted _X_ at line 49. Pseudocode example:

```
var x = 1
x == x + 1
```

The assignment `x == x + 1` does not have to be surrounded by and `if` statement checking that `x` is defined. We did it in the preceeding line, and assume it to be true. And that's a good thing. Having the compiler guarantee such logical constraints as a consequence of ordering relieves us of a lot of typing and mental overhead. We like it so much, we often try to write code the same way even when such guarantees can't be made.

```
async function foo(x) {
    var y = await makeHttpRequest(x)
    y == y + 1
    var result = await writeDatabase(y)
    return result
}

...


if Rec.x > 5 {
    result = await foo(Rec.x)
} else {
    result = false
}
```
The async/await pattern is one of many techniques for making asynchronous code feel the same as old-school sequential code. Sometimes that abstraction works, but it can lead to complexity and make it difficult to reason about your code. What if, while waiting for the response to `makeHttpRequest`, `Rec.x` was changed to `4` by some other thread of execution? Is it still valid to take the response and write it to the database? And what if the database write failed? And so forth. The sequential abstraction, the idea that the code will execute in the order written, only helps us when the implied _logical_ constraint can be enforced as a _temporal_ constraint: if `Y` occurs after `X` in the code, then `Y` is always valid once `X` has executed.

A similar example can occur with user interfaces:

```
if Rec.X > 5 {
    textBox.onchange = async function (newText) { 
                                var x = await makeHttpRequest(newText) 
                                ...
                             }
}
```
Is it still valid to send the result of the `textBox.onchange` event if `Rec.X` changes?

# Unification
