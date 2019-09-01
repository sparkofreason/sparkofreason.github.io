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

For such applications, we no longer have the nice clean situation of a single batch of input, transformed by sequential statements to produce a unique batch of output. There are multiple points of input (client, message queues, responses to requests from DBs or other services), these change over time with the state of the system, with no guarantees of ordering. We can try to force such guarantees, but in real distributed system this almost always leads to undesirable effects such as poor performance.

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

# Observations

## Dynamic Input

The examples above all exhibit _dynamic inputs_, inputs that change with the execution or state of the system. A HTTP request often implies that we expect a response (assuming it isn't "fire and forget"). We provide a callback code which is executed *if* a response is received, and perhaps different code to handle errors. By making the request, we've created a new input to our system, a place where the "outside world" will provide some response. Other examples include event callbacks for UI elements, handlers for message queues, and so forth. Abstractly, these are all the same. 

Let's call all such "dynamic inputs" _requests_.  A request is the abstraction indicating our program can receive some input from the outside world, regardless of the implementation details. A _response_ is that input. Assume the minimum about requests and responsees:

* A request might not receive a response.
* A response is always associated with a specific request instance.
* The order of responses to requests is not guaranteed.
* A request might become invalid while a response is "in flight" (more on this below).

## Stateful

Requests are part of the system state. That state is generally implementation specific, as is what you can interrogate or modify, which can make testing a headache. For HTTP service requests, we might need to inject some sort of mock implementation of the service, which not only responded to requests with the appropriate data, but also verified that requests were made as required. Maybe we'd use Selenium or some other browser mock to drive testing of requests to the user interface, something else to mock message queues, and so forth.

The state describing requests may also depend on program state. A request could be issued based on some condition. Some other request then receives a response that changes state rendering the first request invalid. But we have no control over the response arriving. Once a request is invalid, we need to ensure that the response is ignored, for example, don't execute the code in the callback. Otherwise we open the possibility of nasty race condition bugs, and must test such scenarios directly.

## Coupling

Requests represent interfaces with the outside world, systems that are potentially "far away" in both space and time, and not under our direct control. Coupling can occur in at least two ways

1. Logical coupling, like the specifics of what is sent in an HTTP request or database query;
2. Implementation coupling, such as how your language runtime and the OS wire up the handling of send an HTTP request and dealing with the response, if/when it comes.

Abstractions which allow requests to appear as "normal" sequential code cause such coupling to occur deep in the guts of your code. I feel this is a questionable practice. Say you were building an electronic device, which allowed a user to input numbers, and had other interfaces which maybe sent signals to other devices, etc. You wouldn't build this thing where the keypad and other interface points were buried deep inside the device, requiring to to be disassmbled to be accessed. The same thing applies in code. Burying requests in your implementation makes it more difficult to both reason about and test. These points where we interface with users or other systems are generally critically important, and should be exposed at the boundaries of our system. 

And unlike the electronic box analogy, the interfaces are potentially changing over time. Ideally we should be able to query the system and see exactly what requests are pending. And those requests should be abstracted as just data describing the request, explicitly part of the system state. That allows us to focus on business logic, without getting tangled up in the implementation details of interfacing with users or other external services and systems. The code which handles the actual implementation details of wiring up those requests can be separated from the business logic, and thus replaced ad hoc for testing. 

# R-cubed: Reification of Request/Response

## Example

We will use tic-tac-toe as our example. A human will compete against an AI service via a browser client. The usual rules will apply, and additionally we will allow the human to reset the game if things aren't going well.

## Design

A "component" in an R-cubed system could be anything with persistent state, dedicated business rules, and interfaces. The boundaries of what comprises a component are a design decision, but obvious candidates would be UI clients and stateful microservices. The actor model may be helpful in thinking about components, since the state of the component should only be updated by responses to requests resulting from the business logic applied to the current state. 

Each R-cubed component requires three pieces:

1. Business Logic - Implementation of the logical requirements of your application. For tic-tac-toe, these include determining if the game is over (some player won or tie game), which player moves next, or resetting the game state to start a new match. The business logic also maintains the state, and transformation of that state when a request receives a response. The state in the tic-tac-toe example would include the current game board (which squares are empty, or have an `X` or `O`), and the current set of valid requests. Requests will either be a `MoveRequest` for the current player and each empty square, and a `ResetRequest` when it is the human player's turn.
2. Effectors - Effectors implement effects based on the current business logic state. Effectors must therefore be able to query that state. The effectors for tic-tac-toe will be
    * UI - renders the current UI based on the business logic state, and wires up events for user inputs corresponding to valid requests.
    * AI - calls the AI service to get the next move when it's the computer's turn, handles the response.
3. Data flow - some implementation which allows effectors to 
    * Get updated query results when business logic state is changed
    * Provide responses to requests back to the business logic.

An R-cubed component has the following lifecycle:

1. Initialize business logic state
2. Effectors run queries against current state
3. Effects (rendering UI, HTTP requests, etc.) are performed based on query results.
4. Wait until a response is received from a pending request.
5. Send the response to the business logic.
6. Business logic updates state based on response data.
7. Goto (2) until component is terminated.

Business logic state updates from responses must be serialized, i.e. each update must run to completion before processing another response. Race conditions are thus avoided, as the state changes from a response may invalidate other pending requests.

## Implementation

### Enforcing Logical Constraints

Before diving into the specifics of the tic-tac-toe implementation, we examine some further choices to be made around the implementation of the business logic. Effectors and data flow are pretty much just "plumbing", and you will choose the techniques and implementations best suited for your particular effects, programming language, and runtime requirements. The business logic is often the "money code" (hence the name), and bugs there tend to hurt. Consider the following scenario in our tic-tac-toe game:

1. Human makes a move
2. A request is sent to the AI service to get the computer's move
3. Human realizes they chose badly, and pushes the Reset button before the response is received from the AI.
4. Board resets to a new game
5. Response is received from the AI

What happens at step (5)? The correct action would be to ignore the response, which means we removed or otherwise invalidated the associated `MoveRequest` from the business logic state. We could just be careful to write code to do that explicitly: when Reset is pushed, delete all of the `MoveRequest`'s. What if additional functionality were added to future versions that could invalidate requests to the AI? We'd have to be disciplined enough to delete the `MoveRequest`'s there as well.

Words like "careful" and "disciplined", when applied to programming, imply a high likelihood of creating incorrect code. Why do tools like static type checkers or runtime data validators exist? So you don't have to be careful. You could write code with no type validation whatsoever, try to remember the type of every variable, shape of every compound data structure, signature of every function, and so forth. But why? That's the kind of shit computers are really good at: ensuring constraints are not violated, everywhere and always. Such tools allow us to dedicate more mental energy to the stuff that rings the cash register, implementing the business requirements.

The constraints we're interested in here are not about data per se, but logical statements of the form `if X then assert Y`, where `X` and `Y` are logical assertions based on the business logic state. An example could be `if x == 5 then assert y == -1`. When the value of `x` becomes `5`, we want the state to contain the fact that `y` has the value `-1`, which may imply a state change. If `x` subsequently became `4`, we want to retract the previously asserted fact `y == -1`. Returning to our tic-tac-toe example: `if player == computer AND square_5 == empty then assert MoveRequest(player == computer, square == square_5)`. If the condition becomes false by hitting Reset (or any other cause), the `MoveRequest` will be retracted, thus avoiding the race condition with the AI service response.

The general classification of such functionality is called [logic programming](https://en.wikipedia.org/wiki/Logic_programming). The specific type of logic programming we will employ is [foward chaining rules](https://en.wikipedia.org/wiki/Logic_programming) with [truth maintenance](https://en.wikipedia.org/wiki/Reason_maintenance).

### Maali

[Maali](https://github.com/Provisdom/maali) is a Clojure library, built on the excellent [clara-rules](http://www.clara-rules.org/) library. clara-rules is an implementation of forward-chaining rules with truth maintenancy. Maali adds some functionality which I found useful for R-cubed:

1. clara-rules defaults to using Clojure `defrecord`'s (essentially typed Java objects) to represent fact data. Maali uses Clojure maps, with [Clojure spec](https://clojure.org/guides/spec) used to define fact "types" and perform runtime validation. spec provides more flexible validation than the simple struct-like type constraints of a `defrecord`, and we used it throughout our code for other purposes.
2. clara-rules defines rules as Clojure `var`'s via the `defrule` macro, and groups them under Clojure namespaces. Maali allows collections of rules to be defined under a single `var` with a `defrules` macro. Correspondingly, session creation in clara-rules specifies the namespaces containing the rules included in the session, while in Maali you just provide the rule collection vars. The same applies to queries, e.g. `defquery` -> `defqueries`. (Aside: I think this choice was mostly a matter of personal preference, though it may have facilitated dynamic reloading of rules in ClojureScript, a la figwheel.)
3. Maali provides some specific tooling around R-cubed for wiring up data flow, creating requests, handling responses, and dealing with cancellation.
4. Maali provides a base set of rules for maintaining logical consistency of requests/responses.


