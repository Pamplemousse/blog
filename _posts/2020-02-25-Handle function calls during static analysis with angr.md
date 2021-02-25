---
layout: post
title: Handle function calls during static analysis in angr
tags: [ angr, development, binary analysis ]
---

On the research project I work on at [SEFCOM](https://sefcom.asu.edu/), I use [`angr`](https://angr.io/) to statically analyse binary programs.

Incidentally, I was invited to give a presentation as part of [CSE545](https://cse545.tiffanybao.com/) during the Fall semester of 2020 at [ASU](https://www.asu.edu/).
This talk was meant to be a hands-on introduction on data-flow analysis, using [`angr`](https://angr.io/), to find "taint-style" vulnerabilities <sup id="a1">[1](#f1)</sup> in binaries.
Thanks to the one of the class's TA, the [video recording](https://www.youtube.com/watch?v=4SMRnpuqN6E&start=490) is available.
Furthermore, I published [the slides of the presentation](https://docs.google.com/presentation/d/13SDNRKHblo2xenczp9m6rQahigtwygmUcrBhZ-G3gvo), as well as [the illustrating code examples](https://github.com/Pamplemousse/bits_of_static_binary_analysis/).

Some of the examples do not work "as is".
It means that for the people trying to reproduce it, extra elbow grease is necessary.
Sadly, it can be somewhat of a tedious (and painful) process: [`angr`](https://angr.io/) is not really stable (its API evolves wildly depending on the needs of people working on it), and [documentation](https://angr.io/api-doc/angr.html) is helpful, but not self-sufficient.

This post is aiming to bridge the gap for who would like to get similar examples working.
In particular, by answering the question:

**How to write a function handler to simulate the effect of a function on the state of the analysis?**

This post is divided in four sections:

  * [Context](#context): Presentation of the analysis, and problems encountered;
  * [Usage and description](#usage-and-description): Runthrough of the documentation, implementation requirements and first thoughts;
  * [Examples](#examples): Examples of handlers for a local and an external function;
  * [One step beyond: Inter-procedural analysis](#one-step-beyond-inter-procedural-analysis): Discussion and high-level overview of turning `ReachingDefinitionsAnalysis` inter-procedural using function handlers;
  * [Conclusion](#conclusion): A closing proclamation.

If you already know what "function handler" means, you can skip the [Context](#context) section, and start reading from the [Usage and description](#usage-and-description) section.


# Context

At a high level, we can use a static analysis to gather data-flow facts about the variables of programs without executing them.
To do so, such analysis somewhat sequentially interprets the effects of program's statements on the state it keeps track of <sup id="a2">[2](#f2)</sup>.

#### But what if such a statement is a function call?

Well, the analysis could continue on the statements of the targeted function, and then jump back to where it was once the function returns.

#### ... And what if this function is an external function? For example provided by a dynamically linked library?

Ah! In such case, the statements that make up the content of the targeted function (its implementation in the binary) are not directly available for analysis.

One thing to note is that we don't really want to analyse a external library as part of the process:
We want to focus on the binary at hand, and prefer to avoid spending resources (computing time and memory) tracking what happens "outside" of it...

Most of the time though, we "know" what a library function does.
Here are examples of what we "know" about a couple of libc functions:
  * [`printf`](https://linux.die.net/man/3/printf): Uses several parameters to deterministically compose a string, and write it to `stdout`;
  * [`malloc`](https://linux.die.net/man/3/malloc): Allocates a chunk of memory of size determined from its first parameter, and return a pointer to it;
  * [`strcpy`](https://linux.die.net/man/3/strcpy): Copies the content of its second parameter into the memory area pointed to by its first parameter.

From the program perspective, and thus the analysis perspective, these functions are black boxes: their implementation details remain hidden.
However, we are only interested in the effect such functions have on the state of the system when the program is running;
From the analysis perspective, the effect they have on the representation of this state.

#### So?

What we need in both cases is **a mechanism to produce the effect of a function on the state representation managed by the analysis**.
This is achieved using **function handlers**.

In the first case (local function), a function handler should drive the analysis to the function called, and return adequately;
In the second case (external function), a function handler should update the analysis state respecting the "known" function behavior.


# Usage and description

We are implementing our analysis using `angr`'s `ReachingDefinitionsAnalysis`.
As described in [the documentation](https://angr.io/api-doc/angr.html?highlight=cfg#angr.analyses.reaching_definitions.reaching_definitions.ReachingDefinitionsAnalysis), it takes an optional `function_handler` parameter.

To work, what is passed via `function_handler` needs to inherit from the `FunctionHandler` [astract base class](https://docs.python.org/3/glossary.html#term-abstract-base-class):
As you can see in [the documentation of `FunctionHandler`](https://angr.io/api-doc/angr.html?highlight=cfg#angr.analyses.reaching_definitions.function_handler.FunctionHandler), it means that the given `function_handler` must have the following methods:
  * `hook`: A mean for the handler to have a reference to an analysis, to be able access to information about its context (architecture, facts gathered in the knowledge base, etc.).
    In particular, [`ReachingDefinitionsAnalysis` calls it at initialisation](https://github.com/angr/angr/blob/0558f5758814cf3f17912b0621f4adb8d0f92240/angr/analyses/reaching_definitions/reaching_definitions.py#L101);
  * `handle_local_function`: That the analysis will run when it encounters a call to a local function.

Those are the minimal requirements for a `function_handler` to have.

Then, for `ReachingDefinitionsAnalysis` to be able to deal with say `printf`, `malloc`, or `strcpy`, we would add the corresponding methods: `handle_printf`, `handle_malloc`, and `handle_strcpy` to the concrete class inheriting from `FunctionHandler`.
For example, such a concrete class `MyHandlers`, would produce instances exposing `handle_printf`, that will be called during the analysis when a call to `printf` is encountered in the binary (and respectively `handle_malloc`, `handle_strcpy` for calls to `malloc`, `strcpy`).
<sup id="a3">[3](#f3)</sup>

**To recap**, and because the terminology is somewhat confusing:
  * A "function handler" is a (Python) method that will be called by the analysis when encountering a `call` instruction;
  * `FunctionHandler` is an ABC class that describe what a concrete class (say `MyHandlers`) to have to work with `angr`;
  * `function_handler` is the name of the parameter to pass the `ReachingDefinitionsAnalysis`; It's a kind of `MyHandlers`, and thus of `FunctionHandler`, exposing "function handler**S**";


# Examples

Let's see what it looks like in practice.

## Binary to analyse

We will analyse the binary produced by [command_line_injection.c](https://github.com/Pamplemousse/bits_of_static_binary_analysis/blob/main/source/command_line_injection.c) .
Here is how to download and compile the code:

```bash
git clone git@github.com:Pamplemousse/bits_of_static_binary_analysis.git
cd bits_of_static_binary_analysis
make
```

If everything went fine, running `./build/command_line_injection ~/` should list your home directory.

## The simplest analysis

The most straightforward analysis starting from the [function `main`](https://github.com/Pamplemousse/bits_of_static_binary_analysis/blob/313f93f1fc287a60dbb3e3217c2c4b3a2cbabf9f/source/command_line_injection.c#L11-L14) looks like the following `analysis.py`:

```python
from angr import Project

project = Project('./build/command_line_injection', auto_load_libs=False)
cfg = project.analyses.CFGFast(normalize=True, data_references=True)

main_function = project.kb.functions.function(name='main')
program_rda = project.analyses.ReachingDefinitions(
    subject=main_function,
)

# Do domething with `program_rda`
...
```

However, as is, the analysis is **intra-procedural**: it only runs on the function `main`.
Pleasantly, when executing `python analysis.py`, `angr` warns us with the following `"Please implement the local function handler with your own logic."`; So we know it encountered a `call` to a local function, and he felt helpless.
Poor `angr`.

## Handle local functions

We can improve `analysis.py` to give the `ReachingDefinitionsAnalysis` the necessary `handle_local_function` that will get triggered when analysing `main`, precisely on the instruction calling `check`.

```python
from angr import Project
from angr.analyses.reaching_definitions.function_handler import FunctionHandler


class MyHandler(FunctionHandler):
    def __init__(self):
        self._analysis = None

    def hook(self, rda):
        self._analysis = rda
        return self

    def handle_local_function(self, state, function_address, call_stack, maximum_local_call_depth, visited_blocks,
                              dependency_graph, src_ins_addr=None, codeloc=None):
        function = self._analysis.project.kb.functions.function(function_address)

        # Break point so you can play around with what you have access to here.
        import ipdb; ipdb.set_trace()
        pass

        return True, state, visited_blocks, dependency_graph

project = Project('./build/command_line_injection', auto_load_libs=False)
cfg = project.analyses.CFGFast(normalize=True, data_references=True)

handler = MyHandler()

main_function = project.kb.functions.function(name='main')
program_rda = project.analyses.ReachingDefinitions(
    function_handler=handler,
    observe_all=True,
    subject=main_function
)

# Do domething with `program_rda`
...
```

Running `python analysis.py`, we now get a shell thanks to the breakpoint placed in the `handle_local_function`.
From there, I invite you to investigate and play around with what you can do;
And remember: you have access to a lot of facts gathered by `angr` through `self._analysis.project` whether it be `.arch`, `.kb`, etc.

## Handling external functions

As presented earlier, handlers can also be triggered on calls to library functions, and used to model the effects of code that cannot be directly analysed.
In [our example](https://github.com/Pamplemousse/bits_of_static_binary_analysis/blob/313f93f1fc287a60dbb3e3217c2c4b3a2cbabf9f/source/command_line_injection.c#L7), we can see that the function `check` calls the libc function `sprintf`.

Here is a new `analysis.py` that showcases how to have the analysis to consider this `call`;
With a richer `MyHandler`, containing a `handle_sprintf` method.

```python
from angr import Project
from angr.analyses.reaching_definitions.function_handler import FunctionHandler


class MyHandler(FunctionHandler):
    def __init__(self):
        self._analysis = None

    def hook(self, rda):
        self._analysis = rda
        return self

    def handle_local_function(self, state, function_address, call_stack, maximum_local_call_depth, visited_blocks,
                              dependency_graph, src_ins_addr=None, codeloc=None):
        function = self._analysis.project.kb.functions.function(function_address)
        return True, state, visited_blocks, dependency_graph

    def handle_sprintf(self, state, codeloc):
        # Break point so you can play around with what you have access to here.
        import ipdb; ipdb.set_trace()
        pass

        return True, state

project = Project('./build/command_line_injection', auto_load_libs=False)
cfg = project.analyses.CFGFast(normalize=True, data_references=True)

handler = MyHandler()

sprintf_plt_stub = project.kb.functions.function(name='sprintf', plt=True)
program_rda = project.analyses.ReachingDefinitions(
    function_handler=handler,
    observe_all=True,
    subject=sprintf_plt_stub
)

# Do domething with `program_rda`
...
```

Notice that for the sake of example simplicity, the analysis gets started on the `sprintf` PLT stub reconstituted by `angr`.
If it was not, this example would be hitting the `handle_local_function` first, because `check` has a `call` instruction pointing to a PLT location, which is not at an external address!
In other words, handling external functions that are called using the PLT mechanics, requires to start a `ReachingDefinitionsAnalysis` on the targeted PLT stub, with the proper handler.

Ideally, we would like to start the analysis on the function `check`, and expect the `handle_sprintf` to be called sometime:
In particular, the analysis should use `handle_local_function` to point the analysis at the PLT stub, which in turn should end up triggering the `handle_sprintf`.

Coincidentally, this is a special case of a more generic problem: How to perform inter-procedural analysis?


# [One step beyond](https://www.youtube.com/watch?v=C9N8piRFVcU): Inter-procedural analysis

With real world programs, it is very unlikely that all the responses to analysts' questions are waiting at a shallow level.
Most of the time, we want to start the analysis from the entrypoint of the binary, and expect it to carry on across function calls until we get the information we were looking for.
In our example, this means starting the `ReachingDefinitionsAnalysis` on the `main` function, and expecting it to analyse `check`, as well as calling `handle_sprintf`.

Because we want an analysis to run over multiple functions, we need an **inter-procedural** analysis.
Sadly, this is currently not implemented in `angr` main repository!

[In the presentation](https://docs.google.com/presentation/d/13SDNRKHblo2xenczp9m6rQahigtwygmUcrBhZ-G3gvo/edit#slide=id.g9a7d25ed88_10_3), and [the corresponding video segment](https://www.youtube.com/watch?v=4SMRnpuqN6E&start=2825) I however presented at a "high level" how we can turn `angr`'s `ReachingDefinitionsAnalysis` into an inter-procedural analysis.

The idea is to run it recursively: every time a `call` to a local function is encountered, a "child" `ReachingDefinitionsAnalysis` is started on the targeted function, and, once finished, the analysis state at its end is "copied" back to the parent, for it to continue from (after the `call` instruction).

Its implementation relies on function handlers.
In particular, `handle_local_function` is where the "recursiveness" happens:
  * It starts the child `ReachingDefinitionsAnalysis` on the targeted function, with proper parameters (passing the current `kb`, initialising the child with the parent state using the `init_state` parameter, forwarding the `function_handler`);
  * It updates the parent's `.observed_results` when the child returns, for the parent to be aware of what was captured during the child's run;
  * It returns the `state` (which contains the current `live_definitions`) for the parent to continue from, as well as other structures the analysis records (`visited_blocks`, `dep_graph`).

Some functions can have several exits (in the case of multiple `return` statements in the source for example), and thus several output states from the analysis perspective!
In such case, the `handle_local_function` must merge those states together to create a unique one for the parent analysis to resume from.

# Conclusion

Function handlers are a handy tool for `angr`'s static analysis using `ReachingDefinitionsAnalysis`: they can be leveraged to apply the effect of external function to the state without having access to their implementation.

By applying the same principle on local functions, they even bring us one step beyond: inter-procedural analysis is nothing more than customization of the analysis behavior (recursiveness, state management, and internal bookkeeping) on `call` instructions.

**Hoping you found those examples enlightening, happy hacking!**

---
<b id="f1">1</b> By "tainting" a variable taking a value from a user input, and propagating this taint on use, one can find other variables that can be influenced by a user input. Tainted variables being used for sensitive operations (arguments to `execve`, or `system`, affectation to a buffer of fixed size, etc.) points to potential security vulnerabilities.[↩](#a1)

<b id="f2">2</b> If you want to learn more details about how the analysis works, and a more concrete example of such analysis, I strongly encourage you to go look at the presentation mentioned above, [available on YouTube](https://www.youtube.com/watch?v=4SMRnpuqN6E). [↩](#a2)

<b id="f3">3</b> For those interested in the underlying mechanics on the `angr` side of things, the handler's instance method is called in [angr/analyses/reaching_definitions/engine_vex.py](https://github.com/angr/angr/blob/0558f5758814cf3f17912b0621f4adb8d0f92240/angr/analyses/reaching_definitions/engine_vex.py#L588-L591). [↩](#a3)
