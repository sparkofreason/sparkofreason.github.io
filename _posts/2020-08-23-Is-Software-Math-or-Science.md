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
say, be 90% sure the software won't fail over the input domain. We could do some
more testing and increase probability; or we might do more testing and find a failure case.
The mathematical approach is clearly more desirable. If you could say with certainty that your
software is correct, then the decision to ship it is easy. The scientific approach leaves
that decision in more of a haze, where the choice has to be made in the face of some
uncertainty. Your code might fail in the wild. What are the chances of that, and what is 
associated cost? Can you take that risk, or is more work required to reduce the risk?

People like certainty, so the attraction of formal mathematical methods is not
surprising. Unfortunately, for Turing-complete languages, the mathematical approach
is limited. [Rice's theorem states that 
all non-trivial, semantic properties of programs are undecidable](https://en.wikipedia.org/wiki/Rice%27s_theorem). 
That's a fancy
way of saying you can't generally prove anything interesting about a program written
in a Turing-complete language. Now, you *could* give up Turing-completeness. Total
functional languages forego Turing-completeness for the ability to guarantee that your
program will terminate. With all the hoo-hah about static type analysis, I'm surprised
this approach doesn't get more attention. It seems a lot more compelling than 
"type correctness" given the cost of things like buffer-overrun
vulnerabilities.

But total languages make loops hard to write. The apparent unshakeable popularity
of Turing-complete languages likely is because they make it pretty
easy to write code that does stuff. That implies they also make it easy to write
code that does the *wrong* stuff, hence efforts to bolt on things like static type
systems, where we try to impose some set of constraints that will yield 
mathematically provable statements about your code.

But remember Rice's theorem: we can't generally prove non-trivial semantic properties
of the program. That leaves syntactic properties, in other words, you can only 
prove things about how the program is written, not what it will do when it runs.
Which might go farther than you'd think, see e.g. some of the examples given in
[this article](https://lexi-lambda.github.io/blog/2020/08/13/types-as-axioms-or-playing-god-with-static-types/).
That said, real-world type systems generally go beyond the limits 
of purely provable statements. As discussed in
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

None of this means that static type analysis can't be useful. It just isn't
providing deep mathematic truths about your software. And that brings us back
to the original question: is software correctness math or science? In 
the common case, it's clearly science. THe question and its answer is not
simply academic, but core to what software engineers do
every day. We don't just create software, any more than scientists just
create abstract hypotheses. We also must provide the evidence that our
software is correct. The effort expended using various tools in that
task must be informed by their utility. You wouldn't use a screwdriver to
chop down a tree. Nor should you find yourself using a tool that
makes it difficult to ship code without providing some corresponding
benefit of providing evidence of correctness.

Treating software correctness as science rather than math allows
us to broaden our view of what tools and techniques can be used,
and further allows for consideration of the effectiveness of those
approaches, whether the benefits justify the costs.
A common approach today would be to combine type safety (via static
type analysis by the compiler), unit tests, and code coverage.
Unit tests and code coverage clearly follow the scientific method.
A test is an experiment, covergage tells you for which part of the code
the result of that experiment provides evidence for correctness (or not).
Static analysis a bit of a different flavor, but the end result is the
same, providing evidence that certain classes of errors won't be
encountered at runtime.

Now suppose you were implementing an new monad instance. How would you
verify that your implementation follows the [monad laws](https://wiki.haskell.org/Monad_laws)?
These are clearly semantic properties, statements about what happens when code
executes, not its syntactic structure. Since we're talking about "laws", we ideally
would validate they are true for all possible inputs. But that is rarely
practical. Unit tests as usually employed check one or perhaps a handful
of possibilities, maybe better than nothing, but pretty weak science.

Haskell's answer to this was *property based testing*, as embodied in
[QuickCheck](https://begriffs.com/posts/2017-01-14-design-use-quickcheck.html).
The article states "Of course the only way to truly guarantee properties of programs
is by mathematical proof," but Rice's Theorem tells us this is undecidable, or best
case when we have a finite input domain (say all double-precision floats), the
"proof" would be running the program against every possible input. Property-based
testing instead runs a large-ish subset of possible inputs, and checking that
the results satisfy desired properties, like the monad laws. How large?
Depends on how much evidence you need, combined with prior information about
your code, like the level of evidence of correctness for library functions utilized 
in the code under test. Property-based testing is, at least qualitatively, the
embodiment of the application of [Bayes' Theorem](https://www.mathsisfun.com/data/bayes-theorem.html),
particularly when phrased in the language of science. Given
* *H* is the hypothesis under test, i.e. "program is correct"
* *D* is data gathered in experiments, which we'll generalize to be "observations"
about our program, e.g. static analysis results, test results, etc.
* *I* is prior information

then Bayes' Theorem is written:

>*P(H|D,I) = P(D|H,I) P(H|I) / P(D|I)*

The *P* here means "probability", but should be interpreted more as "belief", the
degree that you believe a statement, expressed as a value between 0 (absolutely false) 
and 1 (absolutely true). 
The vertical bar means given, so in words this reads:

> The belief that your program is correct given the observations and prior information
is equal to the belief that you would make those observations GIVEN your program is correct
and the prior information, times the belief that you thought your program was correct
given ONLY the prior information, divided by the belief you would have made the same observations
regardless of correctness.

The details of what this means and how it might be employed are beyond the scope
of what I want to write here. Indeed, many volumes have been written, and
Bayes' Theorem has been "rediscovered" many times throughout history
(see [The Theorem That Would Not Die](https://www.amazon.com/Theory-That-Would-Not-Die/dp/0300188226)).
This is not some esoteric statistics formula, but the mathematical expression
of *scientific* reasoning. Mathematical reasoning deals with absolutes, statements
are either true or false, and we use axioms and the tools of logic to attempt to prove statements
either way. Science deals with degrees of belief, a spectrum between true and false, informed
by our beliefs and observations. Bayes' Theorem is a key tool in reasoning with degrees
of beliefs, and reflects how people actually think about the real world, where
mathematical purity is rarely applicable.

Let's take a brief qualitative example:
* *H* is the property that our program never crashes with an unhandled exception.
* *I* is some combination of our knowledge about the code, and results of static type analysis
by the compiler.
* *D* is observations we make by running tests, i.e. does the program crash for a given test?

Translating to Bayes' Theorem: The belief that the program never crashes with an unhandled exeception given the test
results and our knowledge of the code and static analysis results is equal to:
* The belief that we would have seen those test results given that our program never crashes
and our prior knowledge/analysis,
* Multiplied by the belief that our program would never crashes given only the prior information,
* Divided by the belief that we would have seen the test results regardless of whether our program
crashes.

Don't overthink the quantitative aspects, like how you will assign numbers to these various
beliefs. It's hard in general, and hasn't really received a lot of attention for
these types of software problems. Instead focus on the qualitative method that follows
for updating beliefs about program correctness, based on whatever *I* and *D* we
might have. The page about Bayes' Theorem linked above has some good examples to give the
flavor, particularly the one about the probability of fire given that you see smoke,
which should have some obvious correspondence with our software example here.


