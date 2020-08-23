---
layout: post
title: 'Is Software Correctness Math or Science?'
excerpt_separator: <!--more-->
published: false
---

The never-ending brouhaha over static vs. dynamic typing strikes me as misguided. 
The broader issue with software is that of *correctness*. Does my program do the things
I require? Does it not do things that would be "bad"? This is where the math vs.
science comes in.

* Math? If software correctness is a purely mathematical question, then "correctness"
is a theorem which we would be able to prove as true or false via application of logic
and some set of axioms.
* Science? If software correctness is a scientific question, then "correctness"
is a hypothesis, for which we gather evidence which updates the weight of belief
in that hypothesis (and correspondingly the weight of the alternative, that the software is
not correct).

<!--more-->

We want to be clear on the distinction, particularly in the outcome. If you can prove
correctness via mathematic methods, you prove a statement of absolute truth, for 
all possible inputs. The scientific approach leaves room for uncertainty. We might,
say, be 90% sure the software won't fail over the input domain. We perhaps good to some
more testing and increase probability; or we might do more testing and find a failure case.
The mathematic approach is clearly more desirable. If you could say with certainty that your
software is correct, then the decision to ship it is easy. The scientific approach leaves
that decision in more of a haze, where the choice has to be made in the face of some
uncertainty.

People like certainty, so the attraction of formal mathematical methods is not
surprising. Unfortunately, for Turing-complete languages, the mathematical approach
is limited. [Rice's theorem](https://en.wikipedia.org/wiki/Rice%27s_theorem) states that 
all non-trivial, semantic properties of programs are undecidable. That's a fancy
way of saying you can't generally prove anything interesting about a program written
in a Turing-complete language. Now, you *could* give up Turing-completeness. Total
functional languages forego Turing-completeness for the ability to guarantee that your
program will terminate. With all the hoo-hah about static type analysis, I'm surprised
this approach doesn't get more attention. Given the cost of things like buffer-overrun
vulnerabilities, it seems a lot more compelling than "type correctness".

But total languages make loops hard to write. The seeming unshakeable popularity
of Turing-complete languages probably stems from the fact that they make it pretty
easy to write code that does stuff. That implies they also make it easy to write
code that does the *wrong* stuff, hence efforts to bolt on things like static type
systems, where we try to impose some set of constraints that will yield 
mathematically provable statements about your code.

But remember Rice's theorem: we can't generally prove non-trivial semantic properties
of the program. That leaves syntactic properties, in other words, you can only 
prove things about how the program is written, not what it will do when it runs.
Which might go farther than you'd think, see e.g. some of the examples given in
[this article](https://lexi-lambda.github.io/blog/2020/08/13/types-as-axioms-or-playing-god-with-static-types/).
That said, real-world type systems generally go beyond the limits implied by
staying in the realm of purely provable statements. As discussed in
[Static Program Analysis](https://cs.au.dk/~amoeller/spa/), type checkers are
generally approximations. And where they need to err, they try to do it on the
conservative side, occasionally failing programs which are semantically correct
but simply do not match the constraints of the type system,
rather than passing programs which will fail. Now you know why you've spent
hours scratching your head why the compiler kicks back code which clearly will
run fine.

Further, that conclusion does not account for the fact that
["Typing is hard"](https://typing-is-hard.ch/). First, devising 
a mathematically rigorous type calculus is difficult, moreso
considering it has to work nicely with the actual programming
language. Scala's type checker, for instance, is mathematically both 
* Unsound:  Will accept incorrectly typed programs
* Undecidable: Cannot determine type correctness for all programs in finite time

Add to that the type checker is just another program.
Who's verifying that program is correct? Can that even be verified
in general?
