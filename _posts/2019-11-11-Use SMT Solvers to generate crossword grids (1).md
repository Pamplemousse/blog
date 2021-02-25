---
layout: post
title: Use SMT Solvers to generate crossword grids (1)
tags: [ development z3 ]
---

This post is part of a series on using SMT Solvers to generate crossword grids.

  * Introduction to SMT, and programming with SMT Solvers (**currently reading**);
  * <a href='/2019/11/12/Use-SMT-Solvers-to-generate-crossword-grids-(2).html' target='_blank'>Definitions and first formulas</a>;
  * <a href='/2019/11/13/Use-SMT-Solvers-to-generate-crossword-grids-(3).html' target='_blank'>Plumbing everything together, complete formula, and results</a>.

Thanks [@geistindersh](https://twitter.com/geistindersh) for his feedback, and corrections!

---

SMT solvers are tools that are used in several fields. By modeling complex problems into logical formulas, and then leveraging the power of a Solver hoping to find values satisfying these formulas, it is possible to obtain solutions for the targeted problems.

When I first encountered this approach in a class on program analysis [^1], the whole concept of encoding problems with mathematics was not very straightforward, and required a bit of mental gymnastics.

However, after some practice, I became more accustomed to this idea, and recently had the opportunity to exercise and study these approaches more in depth.
Now that it feels familiar, I believe it's the perfect time to write what I wish I could have read earlier.

Hence, in the following blog posts, we will explore the use of SMT Solvers in a recreational way, making use of them to solve an absolutely unimportant problem: **generation of crossword grids**!


# Introduction

## SMT: Satisfiability Modulo Theory

What is SMT by the way?

> SMT problem is a decision problem for logical formulas with respect to combinations of background theories
[^2]

Huh? Let's break that down:
  * A **decision problem** is a question that can be answered by `Yes`, `No`, or `Don't Know` [^3];
  * **Logical formulas** are "mathematical" formulas using variables and operations; Such formulas can be evaluated to `True`, or `False` (depending on the values that the variables take, and the considered operations);
  * **Background theories** can be thought as the "universe in which our formula lives". Some examples:
      - Booleans: variables can take the values `True` or `False`, and operations being $$ \land, \lor, \lnot $$ (respectively logical and, or, not) ...
      - Integers: variables can take integers values $$ \{ -\infty, ..., -2, -1, 0, 1, 2, 3, ... \infty \} $$, and operations being arithmetic operations: $$ +, -, *, /, \%, ... $$
      - Strings: variables are sequences of characters, and operations can be substring comparison, concatenation, ...

In our context, we will consider the latest, because crosswords involve finding words respecting certain constraints.

But before explaining how we will use this particular theory, we need to explicit one can generally use constraints Solvers to help solving problems.


## Solvers

SMT Solvers (also known as constraints Solvers, or theorem Provers) are computer programs, that take formulas expressed under a specific theory as input, and answers one of the following:
  * **`SAT`**isfiable, if there exists a set of values for (a valuation of) the variables that make the given formula `True`;
  * **`UNSAT`**isfiable, if there are no values for which the formula is `True`; In other terms, the formula will always evaluate to `False` no matter how hard we try;
  * **`Don't Know`**, if the Solver did not manage to give one of the previous result under a specific time bound.

We will assimilate SMT Solvers as magical [^4] black boxes, their inner working remaining mysterious.
We can feed formulas to them, and expect one of the above response.

$$
  formula
  \xrightarrow[]{\text{SMT Solver}}
  \begin{cases}
    \text{SAT} \\
    \text{UNSAT} \\
    \text{Don't Know}
  \end{cases}
$$

And if the formula is `SAT`, the Solver will return a proof alongside its answer: a set of values for the variables appearing in the formula.
To verify the satisfiability using the proof, we evaluate the formula with the given values, and ensures that the computed result is `True`.


#### Z3, the theorem Prover

Z3 is an open source SMT Solver developed by Microsoft Research, and [available on GitHub](https://github.com/Z3Prover/z3).

Well-know, well-documented, and relatively famous, it even comes with bindings for many popular programming languages, and notably Python (in other words, Python code can interact with Z3).

It is for this practical reasons that we will generate our crossword grids using Z3, but note that it could be done with any other Solver supporting a theory for Strings.


## Constraint programming

Now that we know what SMT Solvers are, we can make use of them to create programs to help us find solutions to our problems.

However, that is where the conceptual difficulty arise:
As educated humans, we have been taught and trained to solve problems that we are given.

Yet, here, we want to avoid doing so: Instead, we want to **let a Solver do the hard work** of "understanding" the problem and finding a solution to it.
In other words: We **don't try to construct an algorithm to solve the problem**, but rather, we express the constraints of the problem in Mathematical terms, with logical expressions using variables taking values in an adequate theory.


### Example

Let's consider the following puzzle:

> Find $$ x $$ a String such that: $$ x $$'s first letter is an 'H' and its last letter is a 'S'.

Natural "bad" approach, thinking out loud:

> Educated you: I need to construct $$ x $$ a string. <br/>
> Educated you: First letter is an 'H', so $$ x $$ should look like 'H...' . <br/>
> Educated you: Then, last letter is an 'S'.
> Educated you: So $$ x $$ is anything of the form 'H...S' . <br/>
> Educated you: ... \*thinking really hard\* ... <br/>
> Educated you: Hey! 'HS' works!

"Good" approach, from the constraint programming point of view, thinking out loud again:

> You: I have the following constraints: $$ x $$ is a String, $$ x $$ starts with an 'H', and $$ x $$ ends with an 'S'. <br/>
> You: Hey Solver! Can you give me a value for the following formula: $$ x \text{ is a String } \land x \text{ starts with an 'H' } \land x \text{ ends with an 'S'} $$? <br/>
> Solver: Let me see... <br/>
> Solver: ... \*thinking\* ... <br/>
> Solver: It's `SAT`, and $$ x = \text{'HYPOTHALAMUS'} $$ works! <br/>
> You: Thank you Solver. <br/>
> Solver: No worries.


Solvers are very powerful:
We can use them to avoid dealing with tedious logic and building complicated algorithms.


**In the next blog post, we are going to explicit how we model the crossword grids, to get a Solver to help generate them.**

---
[^1]: <a href='https://www.u-bordeaux.fr/formation/2018/PRMA_68/informatique/enseignement/FRUAI0333298FCOEN_7296/verification-de-logiciels' target='blank'>Software Verification</a>, of the <a href='https://mastercsi.labri.fr/' target='blank'>CSI Masters</a> .

[^2]: Source: <a href='https://en.wikipedia.org/wiki/Satisfiability_Modulo_Theories' target='blank'>https://en.wikipedia.org/wiki/Satisfiability_Modulo_Theories</a> .

[^3]: One of the biggest upset in Computer Science is that not all problems can be "solved" by algorithms: <a href="https://en.wikipedia.org/wiki/Undecidable_problem" target="blank">https://en.wikipedia.org/wiki/Undecidable_problem</a> , but we don't know which ones for sure.

[^4]: Because their implementation is way out of the scope of this post, it can be easier to imagine them as transcendental entities, or at least being beyond our comprehension.
