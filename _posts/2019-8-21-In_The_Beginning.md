---
layout: post
title: In the Beginning, There Were Programs
excerpt_separator: <!--more-->
---

# Order

In the beginning, there were programs. Programs generally consisted of statements which ran in order. The order was originally an actual physical ordering: instructions on a sequential tape, or a sequence of punch cards. Order later became order of lines or "statements" in a text file. Some code may be conditionally executed, and we could jump around with `GOTO`'s, loops, subroutines, functions, etc. But programs basically consisted of chunks of statements executed in order.

<!--more-->

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

The examples above all exhibit _dynamic inputs_, inputs that change with the execution or state of the system. A HTTP request often implies that we expect a response (assuming it isn't "fire and forget"). We provide a callback function which is executed *if* a response is received, and perhaps different code to handle errors. By making the request, we've created a new input to our system, a place where the "outside world" will provide some response. Other examples include event callbacks for UI elements, handlers for message queues, and so forth. Abstractly, these are all the same. 

Let's call all such "dynamic inputs" _requests_.  A request is the abstraction indicating our program can receive some input from the outside world, regardless of the implementation details. A _response_ is that input data. Assume the minimum about requests and responsees:

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
2. Implementation coupling, such as how your language runtime and the OS wire up sending an HTTP request and dealing with the response, if/when it comes.

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

[Maali](https://github.com/Provisdom/maali) is a Clojure library, built on the excellent [clara-rules](http://www.clara-rules.org/) library. clara-rules is an implementation of forward-chaining rules with truth maintenance. Maali adds some functionality which I found useful for R-cubed:

1. clara-rules defaults to using Clojure `defrecord`'s (essentially typed Java objects) to represent fact data. Maali uses Clojure maps, with [Clojure spec](https://clojure.org/guides/spec) used to define fact "types" and perform runtime validation. spec provides more flexible validation than the simple struct-like type constraints of a `defrecord`, and we use it throughout our code for other purposes. Spec doesn't play well with `defrecord` either.
2. clara-rules defines rules as Clojure `var`'s via the `defrule` macro, and groups them under Clojure namespaces. Maali allows collections of rules to be defined under a single `var` with a `defrules` macro. Correspondingly, session creation in clara-rules specifies the namespaces containing the rules included in the session, while in Maali you just provide the rule collection vars. The same applies to queries, e.g. `defquery` -> `defqueries`. (Aside: I think this choice was mostly a matter of personal preference, though it may have facilitated dynamic reloading of rules in ClojureScript, a la figwheel.)
3. Maali provides some specific tooling around R-cubed for wiring up data flow, creating requests, handling responses, and dealing with cancellation.
4. Maali provides a base set of rules for maintaining logical consistency of requests/responses.

The Clojure DSL used to define rules and queries in Maali is largely the same as that in the underlying clara-rules, so most of the [clara-rules documentation](http://www.clara-rules.org/docs/firststeps/) applies. Differences will be highlighted in the code examples below.

### R-cubed in Maali

```clj
(defrules
  ...
  [::move-request!
   "If the game isn't over, request ::Moves from the ::CurrentPlayer for
    eligible squares."
   [:not [::GameOver]]
   [::CurrentPlayer (= ?player player)]
   [?moves <- (acc/all) :from [::Move]]
   [::common/ResponseFunction (= ?response-fn response-fn)]
   =>
   (let [all-positions (set (range 9))
         empty-positions (set/difference  all-positions (set (map ::position ?moves)))
         requests (map #(common/request {::position % ::player ?player} ::MoveResponse ?response-fn) empty-positions)]
     (apply rules/insert! ::MoveRequest requests))]

  [::move-response!
   "Handle response to ::MoveRequest by inserting a new ::Move and switch
    ::CurrentPlayer to the opponent."
   [?request <- ::MoveRequest (= ?position position)]
   [::MoveResponse (= ?request Request) (= ?position position) (= ?player player)]
   [?current-player <- ::CurrentPlayer]
   =>
   (rules/insert-unconditional! ::Move {::position ?position ::player ?player})
   (rules/upsert! ::CurrentPlayer ?current-player assoc ::player (next-player ?player))]

  ...
)
```

Here are the rules used for request/response for the tic-tac-toe moves. The full code can be viewed in the [maali-simple](https://github.com/Provisdom/maali-simple/blob/master/src/provisdom/simple/rules.cljc) repository. The `defrules` macro accepts one or more rule definitions. A rule definition is a vector defined as

1. The rule name. Clojure's namespaced keywords are handy here, since rule names must be unique. I like suffixing rule names with `!` because they represent conditional state transitions. Queries simply bind data with no state change, so no `!` on query names.
2. An optional doc string.
3. One or more clauses (the `if` part of the rule)
4. The symbol `=>` (read as `then` or `implies`).
5. Assertions (or general effects, more on that below.)

The `::reset-board-request!` rule is read as

* *IF*
    * No fact of type `::GameOver` exists;
    * AND Bind the value of the `::player` attribute of the `::CurrentPlayer` entity to the `?player` variable;
    * AND Bind all of the existing `::Move` entities to the `?moves` variable;
    * AND Bind the `?response-fn` variable to the value of the `:response-fn` attribute on the `::common/ResponseFunction` entity.
* *THEN*
    * Conditionally insert a new `::MoveRequest`'s (constructed using the `common/request` function) for all squares not containing a move.

Note that `common` is an alias for the `provisdom.maail.common` namespace, and `rules` aliases `provisdom.maali.rules`. The `::` syntax in Clojure is shorthand for "use the current namespace on this keyword", so `::move-request!` expands to `:provisdom.simple.rules/move-request!`. The stuff around `::common/ResponseFunction` is a convenience for wiring up the data flow dynamically, basically allows effectors to provide a response to a specific request without having to know about the details of the rules code.

The `then` part of our rule inserts `::MoveRequest`'s for the empty squares and the current player. These fact entities are maps with an associated spec `::MoveRequest` that serves as the "type" of the entity. The `insert!` function in clara-rules does not require the entity type to be explicitly stated. For `defrecords`, the type is just part of the object metadata. The Maali functions which manipulate fact entities require the entity type to be explicitly specified, and the provided data is validated against the spec.  The insert in this case is _conditional_, which means that it is subject to truth maintenance. If bindings/conditions in the `if` clauses were to change, then the `::MoveRequest`'s would be automatically retracted. Had we inserted this fact directly into the session (without it being part of a rule), or had we used the `insert-unconditional!` function, the fact would not be managed by truth maintenance.

The conditional vs. unconditional point is important in R-cubed. Requests will usually (always?) be conditional, reflecting the dependence of the valid inputs on the business logic state. Responses, however, come from the "outside world". Responses don't depend on anything in your business logic, and are _unconditional_, inserted directly into the rules session working memory. The logical consequence is that facts asserted as part of a response rule are unconditional as well. Look at the `::move-response! rule:

* *IF* we can
    * Bind the request of type `::MoveRequest` to the `?request` variable, and the `::position` attribute of the `::MoveRequest` to `?position`; 
    * AND find a `::MoveResponse` whose `::common/Request` attribute matches `?request`, `::position` attribute matches `?position`, and binds `?player` to `::player`;
    * AND Bind `?current-player` to the `::CurrentPlayer` fact entity;
* *THEN*
    * Do an unconditional insert of a `::Move` for the `?position` and `?player` specified in the response;
    * AND Update the `::CurrentPlayer` entity to be the other player.

The `::MoveResponse` is the input received from either the human or AI service. The resulting `::Move` fact is thus necessarily unconditional, because its existence and value depends *only* on the response, not on the rest of the state. To further that point: once we're done with the `::MoveResponse`, we're going to delete it (we'll see how below), because we don't want stale input hanging around. If `::Move` were logically dependent on `::MoveResponse`, then deleting `::MoveResponse` would cause truth maintenance to retract the `::Move`, and the game would never progress past an empty board. A benefit of using forward-chaining rules in R-cubed is the explicit specification of which data is a logical consequence of the current state, and what follows from external input.

The `upsert!` on `::CurrentPlayer` is similar. `::CurrentPlayer` starts as an initial fact inserted into the session when the game is started, and so is unconditional. The `upsert!` function is essentially a shortcut for the following:

```clj
(rules/retract! ::CurrentPlayer ?current-player)
(rules/insert-unconditional! ::CurrentPlayer (assoc ?current-player ::player (next-player ?player)))
```

We don't bother suffixing `rules/upsert!` with `-unconditional`, because logically all upserts *must* be unconditional. Here's what happens `insert-unconditional!` above were replaced with `insert!`:

* The current value of the `::CurrentPlayer` fact is bound to `?current-player`.
* `?current-player` is retracted.
* A new value of `::CurrentPlayer` is inserted.
* Truth maintenance sees that the `?current-player` binding for the `if` clauses in this rule has changed, and so retracts the newly inserted `::CurrentPlayer` value.

So `upsert!` is always unconditional. To be otherwise would lead to logical contradictions in your rules, essentially stating "`X` implies `not X`".

We noted above that requests are conditional, and subject to truth maintenance. The `::MoveRequest`'s created by the `::move-request!` will be automatically retracted whenever the `if`-bindings modified:

* The value of the `::CurrentPlayer` changes
* OR The current set of `::Move`'s changes

(Technically, also add OR the `::ResponseFunction` changes, but that shouldn't happen). So, when the `::move-response!` rule fires, all of the (now stale) `::MoveRequest` facts will be retracted automagically by truth maintenance, which saves us some bookkeeping code. `::MoveResponse` however is unconditional, and unless we explicitly retract it, is going to hang around. This is a memory leak if nothing else, and could cause further problems by spuriously firing rules with old input. We could be "disciplined" and explicitly retract the response fact in any rule that processes responses, but that's just asking for bugs.

Instead we can include rules that establish the logical relationships between requests and responses. From `provisdom.maali.common`:

```clj
;;; Common rules for request/response logic.
(defrules rules
  [::cancel-request!
   "Cancellation is a special response that always causes the corresponding
    request to be retracted. Note that the ::retract-orphan-response! rule
    below will then cause the cancellation fact to also be retracted."
   [?request <- ::Cancellable]
   [::Cancellation (= ?request Request)]
   =>
   (rules/retract! (rules/spec-type ?request) ?request)]

  [::retract-orphan-response!
   "Responses are inserted unconditionally from outside the rule engine, so
    explicitly retract any responses without a corresponding request."
   [?response <- ::Response (= ?request Request)]
   [:not [?request <- ::Request]]
   =>
   (rules/retract! (rules/spec-type ?response) ?response)])
```

We won't discuss `::cancel-request!` here. `::retract-orphan-response!` is the rule of interest:

* *IF* we can
    * Find a `::Response` fact with an associated `::Request` attribute bound to `?request`;
    * AND there exists no `::Request` entity in the state with the value `?request`;
* *THEN*
    * Retract the response.

This covers two scenarios:
* We processed a response, which changed the state such that the request was retracted. `::retract-orphan-response!` ensures that the response entity is also retracted.
* Something else invalidated the request while the response was "in flight" from an external source, in which case when the effector inserts the response it gets automatically removed, avoiding potential race conditions.

### Effectors

A key part of R-cubed is the separation of the code performing effects from the logic requesting those effects. This likely requires some "disclipline". Most programming languages don't provide a way to guarantee that business logic code contains no effects. Maali (and the underlying clara-rules) allows you to write arbitrary Clojure code in the `then` part of the rule, so you could do all kinds of stuff there, like mutating the UI, putting stuff in a database, etc. Don't do that. Restrict the `then` clauses to the following (for Maali, or equivalent operations in your language)

* `insert!`
* `insert-unconditional!`
* `retract!`
* `upsert!`
* Any calculations which are functionally pure
* Debug output, like `println`, which doesn't affect the state of the business logic or effectors.

Everything else goes in effectors. Business logic remains functionally pure, which will enable reasoning and testing.

The [view effector](https://github.com/Provisdom/maali-simple/blob/master/src/provisdom/simple/view.cljs) for tic-tac-toe is responsible for changing the view in response to business logic state, as well as wiring up user input and providing any input as responses. An excerpt is shown below:

```clj
...

(defn click-handler
  [request]
  (when request
    (let [response (select-keys request [::simple/player ::simple/position])])
      #(common/respond-to request response)))

;;; Markup and styling from https://codepen.io/leesharma/pen/XbBGEj

(defn tile
  [session position]
  (let [c (click-handler (rules/query-one :?request session ::simple/move-request :?position position :?player :o))
        marker (rules/query-one :?player session ::simple/move :?position position)
        win? (rules/query-one :?winning-square session ::simple/winning-square :?position position)]
    [:td {:id (str position)
          :class (cond-> "tile"
                   win? (str " winningTile")
                   c (str " clickable"))
          :on-click c}
     (condp = marker
       :x "X"
       :o "O"
       "")]))

...
```

The `click-handler` function is going to provide the response to a user click on an empty square. The `::MoveRequest` and `::MoveResponse` specs are defined as (from `provisdom.simple.rules)`:

```clj
...

(s/def ::player #{:x :o})
(s/def ::position (s/int-in 0 9))

...

(def-derive ::MoveRequest ::common/Request (s/keys :req [::position ::player]))
(def-derive ::MoveResponse ::common/Response (s/keys :req [::position ::player]))

...
```
`def-derive` is a macro defined in `provisdom.maali.rules` which defines a Clojure spec and establishes an "is a" relationship with some other spec (via Clojure's `derive` function). So a `::MoveRequest` is an instance of `::common/Request` with additional attributes `::player` and `::position`. That hierarchy proves handy sometimes, in particular allowing us to write a generic `::retract-orphan-response!` rule that applies to all entities deriving from `::common/Request` and `::common/Response`. 

The `common/respond-to` function is a helper that allows effectors to be ignorant of the data flow wiring. All that is required is to call it with the request and response, assuming the request was created with the `common/request` function as we showed above. We don't show it here, but this will insert the response, fire the rules, and alert the effectors that the business logic state has changed.

The `tile` function is responsible for the rendering and event-handling (when relevant). `tile` is called from another function which is rendering the board as an HTML table, and so returns the markup (as [hiccup](https://github.com/weavejester/hiccup/wiki/Syntax)) for a `<td>` element. The `session` argument contains the business logic state, and `position` is the board position of the square being rendered. `provisdom.maali.rules/query-one` is a convenience method which executes a query against the business logic state and returns the first result (useful when you know there will only ever be a single result). The relevant queries for `tile` are (from `provisdom.maali-simple.rules`):

```clj
(defqueries queries
  [::move-request [:?position :?player] [?request <- ::MoveRequest (= ?position position) (= ?player player)]]
  [::move [:?position] [?move <- ::Move (= ?position position) (= ?player player)]]
  [::winning-square [:?position] [?winning-square <- ::WinningSquare (= ?position position)]]
  ...
)
```
The `::move-request` query is parameterized by `?position` and `?player`, so we pass in the position argument supplied to `tile`, and `:o` for player, since the human player is always "O" in this game. If the session contains a `::MoveRequest` for the specified `position` (empty square) AND it is the human's turn to play, `rules/query-one` will return that `::MoveRequest`; otherwise it returns `nil`. `tile` similarly executes the queries for other relevant facts, providing information to render the `<td>` with conditional CSS classes and event handling.

The core AI effector code is

```clj
   (when (not-empty (rules/query-partial session ::simple/move-request :?player :x))
     (let [moves (map :?move (rules/query-partial session ::simple/move))
           board (simple/squares->board moves)
           next-move (ai-fn board)
           {::simple/keys [position player] :as move-request} 
              (rules/query-one :?request session ::simple/move-request :?position next-move :?player :x)]
       (common/respond-to move-request {::simple/position position ::simple/player player})))
```

The pattern is similar to the view. We check if there are `::MoveRequest`'s for the computer (`provisdom.maali.rules/query-partial` allows query execution specifying only a subset of query arguments). If so, we gather up all the existing moves, construct the board configuration, and pass that to the AI service to get the next move. We then find the `::MoveRequest` for thoe AI's chosen move, and respond appropriately.

### Testing

The purpose of testing software is to inform our beliefs about "correctness", that it performs according to our specifications. The discussion of how R-cubed enables testing probably warrants its own long discussion. We'll hit a couple of high points here.

First, remember that while forward-chaining and truth maintenance simplifies R-cubed implementations, they aren't strictly required. But we accrue an additional benefit by declaring business logic as rules. Looking at the rule definitions, they very much resemble the sort of conditional statements employed in various testing scenarios. Suppose that we weren't using rules, and instead of having the `::move-request!` rule we had a `move-requests` function which took in the current state and returned the valid move requests. We then might write some unit tests for `move-requests`, supplying various test-cases, and then checking the output with conditional statements. Those conditions would appear very much the same as the rule definition. This is key: forward chaining with truth maintenance implies that the runtime guarantees the assertions to follow from the conditions, at least at the scope of a single rule. So we don't need to test that directly. Now, it may be that when combined with other rules, you induce some logical contradiction. Using rules doesn't get you out of testing, but it does provide some guarantees at a lower level, and allows more testing effort to be spent at a level more like "integration testing", where nastier bugs are likely to surface. Such bugs are often the result of logical contradictions between rules, which tend to be very obvious, for example, when firing the rules truth maintenance won't converge, but bounces back and forth between the offending rules.

Rules aside, R-cubed facilitates direct testing of the business logic. Effects have been completely removed from the business logic. The requests for effects are data, contained in the business logic state, and where applicable responses are just supplied as data. The implementation specifics of the effects are decomplected. The connection between business logic and effectors can (should?) be set at runtime, as determined by your data flow. Swapping out effector implementations for test mocks (or any alternative implementation) is therefore straightforward, without requiring we drag in dependency injection, mocking frameworks, and the like. The business logic itself is intrinsically synchronous and pure. Asynchronous handling is pushed to the effectors. For any state, we can ask for the set of pending requests, and assuming we have sufficient metadata describing the expected responses, can randomly and automatically provide response data. Some example test code for tic-tac-toe is shown below:

```clj
;;; TESTING

(defn check-invariants
  [session]
  (let [moves (map :?move (rules/query-partial session ::simple/move))
        move-requests (map :?request (rules/query-partial session ::simple/move-request))
        game-over (rules/query-one :?game-over session ::simple/game-over)
        winner (rules/query-one :?player session ::simple/winner)]
    (let [counts (into {} (map (juxt key (comp count val)) (group-by ::simple/player moves)))
          xs (or (:x counts) 0)
          os (or (:o counts) 0)]
      ; Make sure we don't have any extra moves. :x goes first so should be
      ; either one ahead or equal to :o.
      (when (or (< 1 (- xs os)) (> 0 (- xs os)))
        (throw (ex-info "Invariant violation: extra moves" {:counts counts}))))
    ; If all the squares are full, the game should be over.
    (when (= 9 (count moves))
      (when (not game-over)
        (throw (ex-info "Invariant violation: game should be over" {}))))
    ; Smart AI should never lose
    (when smart-ai
      (when (= :o winner)
        (throw (ex-info "Invariant violation: smart AI should never lose" {}))))
    ; In teaching mode, the user should never lose
    (when teaching-mode
      (when (= :x winner)
        (throw (ex-info "Invariant violation: human should never lose in teaching-mode" {}))))
    (let [mr-pos (set (map ::simple/position move-requests))
          m-pos (set (map ::simple/position moves))]
      ; Can't request a move for a square that's already been used.
      (when (not-empty (set/intersection mr-pos m-pos))
        (throw (ex-info "Invariant violation: moves and move requests should not have overlapping positions" {}))))))

(defn abuse-simple
  [session-atom iterations]
  (add-watch session-atom :check-invariants
             (fn [_ _ _ session]
               (check-invariants session)))
  (loop [i 0]
    (if (< i iterations)
      (do
        (if (rules/query-one :?game-over @session-atom ::simple/game-over)
          ; If the game is over, just reset
          (let [req (rules/query-one :?request @session-atom ::simple/reset-request)]
            (if req
              (common/respond-to req)
              (throw (ex-info "Should have reset request for game over" {}))))
          ; if the game is not over, then play
          (if (> 0.01 (rand))
            ; Once in awhile, be a jerk and reset the game.
            (when-let [req (rules/query-one :?request @session-atom ::simple/reset-request)]
              (common/respond-to req))
            (let [reqs (rules/query-partial @session-atom ::simple/move-request)]
              (if (not-empty reqs)
                ; If legal moves exist, choose one at random
                (let [{::simple/keys [position player] :as req} (:?request (rand-nth reqs))]
                  (common/respond-to req {::simple/position position ::simple/player player}))))))
        (recur (inc i)))
      (remove-watch session-atom :check-invariants))))
```

`check-invariants` are conditions we want to test. Most of these are for additional app-level features we didn't discuss here. For example, the final implementation of tic-tac-toe allows the user to switch between "dumb" and "smart" modes, which either randomly selects a move or does an exhaustive search of all possible future game moves and chooses the best one. Smart mode should never lose, if the AI is correctly implemented.

`abuse-simple` runs the tests for the specified number of iterations. Each iteration checks the game state and then randomly chooses a request for which a response is provided. The `add-watch` statement at the top of `abuse-simple` subscribes to changes in the business logic state, and calls `check-invariants` for every change. We are simulation testing at the application level, rather than the function level. 

We can further expand the scope of integration testing to include effectors, if/when it makes sense. A good example would be to connect the view effector, let it run, and watch for exceptions, indications of rendering errors, or whatever. [This video shows the results applied to tic-tac-toe](https://www.loom.com/share/b4f27647e35b4447b71d6a2ba2b14612). About the first half of the video has a longish delay between iterations, while the second half reduces this. We can take it even further, and interact with the UI while the simulation is running, as shown in [this video](https://www.loom.com/share/2acfcc211f4f49bb99f06dee4f3176c6).

The tic-tac-toe application doesn't require any sort of generation of response data. The only input is click events (or equivalently the move choice from the AI service), and so the request data completely determines the response. The Maali implementation of TodoMVC does utilize generation of response data based both on response specs and request data:

```clj 
(def request->response
  {::todo/EditRequest             ::common/Response
   ::todo/UpdateDoneRequest       ::todo/UpdateDoneResponse
   ::todo/RetractTodoRequest      ::common/Response
   ::todo/CompleteAllRequest      ::common/Response
   ::todo/RetractCompletedRequest ::common/Response
   ::todo/VisibilityRequest       ::todo/VisibilityResponse})

... 

(defn gen-response
  [request]
  (let [response (sg/generate (s/gen (request->response (rules/spec-type request))))]
    (assoc response ::common/Request request)))

(defn gen-visibility-response
  [visibility-request]
  (let [response (gen-response visibility-request)]
    (assoc response ::todo/visibility (sg/generate (s/gen (::todo/visibilities visibility-request))))))
```

`gen-response` will generate response data based on the spec for the `::Response` entity expected for a given `::Request` spec. `gen-visibility-response` picks a visibility filter based on the valid options in the `::VisibilityRequest`. Those options change with the business logic state, so must be generated dynamically, rather than from a static spec. [Here is another video, showing the simulation test for TodoMVC connected to the view effector](https://www.loom.com/share/8f4f44477fb14d138971695f6861b6be).

# Links

* [Reification of Request and Response (R^3)](https://sparkofreason.gitbooks.io/maali/content/) - A GitBook with some more in-depth discussion of the ideas, as well as a detailed description of the tic-tac-toe application and some extensions.
* [https://github.com/Provisdom/maali](https://github.com/Provisdom/maali) - Maali code.
* [http://www.clara-rules.org/](http://www.clara-rules.org/) - clara-rules site.
* [https://github.com/Provisdom/maali-simple](https://github.com/Provisdom/maali-simple) - tic-tac-toe code based on Maali.
* [https://github.com/Provisdom/maali-todomvc](https://github.com/Provisdom/maali-todomvc) - TodoMVC implementation based on Maali.