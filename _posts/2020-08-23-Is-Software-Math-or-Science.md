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
