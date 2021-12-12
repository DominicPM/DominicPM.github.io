---
title: "Verifying C code with CBMC: an introduction"
date: 2021-04-28T19:17:38+01:00
draft: true
---

## Introduction

The C programming language is all-pervasive, especially in contexts where performance or direct interaction with hardware is necessary.
It's therefore probably not a surprise that Arm &mdash; as a hardware design company &mdash; produces lots of C code: we have teams working on Linux kernel enablement for our architecture, we produce our own operating systems and security monitors, we produce our own TLS library, and so on and so forth.
Many of our C codebases are extremely security sensitive, and within the Systems Research group within Arm Research we've worked alongside product teams to deploy new methodologies to try to spot security problems in our codebases early.
One such methodology is the use of formal methods, specifically *software verification*.

Deploying software verification methodologies in a commercial setting is a challenge in itself, and probably worthy of its own post.
To start, many commercial C codebases are not written in pure C, but use a mixture of C with embedded assembly language, either for performance reasons or to implement features that cannot be expressed in C itself.
This can make many tools choke.
Even worse, many verification tools, especially those originating in academia, tend not to support the full C standard and will fail to parse real C codebases, or fail to reason about C language features that are common in idiomatic code. 
Finally, many verification techniques require properties be expressed in a dedicated *assertion language*, for example formulae from some dedicated logic capable of capturing C states for pre- and postcondition reasoning, or similar.
This tends to make tools hard to deploy in commercial settings if time-strapped product engineers must learn some domain-specific language in order to successfully use a tool. 

## CBMC

The C Bounded Model Checker (or CBMC, henceforth) is a software verification and static analysis tool that we have deployed to good effect internally within product groups in Arm.
After experimentation with a number of tools, we found CBMC to be well suited to our verification tasks, with many advantages over other systems:

- CBMC uses C language expressions to express properties.
No separate assertion language is necessary: engineers can reuse existing C knowledge when writing properties to check.
- CBMC is capable of producing counterexamples when a property is invalidated.
This helps engineers narrow down the source of a bug quickly.
- The CBMC tool is robust, and is capable of parsing and reasoning about real C codebases with the full range of C language features.
Moreover, there are straightforward mechanisms for dealing with mixed C and assembly codebases.
- Finally, perhaps specific to Arm, CBMC uses model checking techniques that are already familiar to many of our product engineers, as model checking is extensively used for verifying microprocessor RTL within the company.

In this series, I intend to introduce C language software verification techniques using CBMC.
I'll start with the very basics in this post.

## Installing CBMC

To install:

- On MacOS, CBMC can be installed through Homebrew using `brew install cbmc`.
Note that at the time of writing, however, CBMC installation through Homebrew
- On Ubuntu Linux, CBMC can be installed through `apt-get` using `apt-get install cbmc`.
- For other platforms,  

## Basics

To test C functions, one might identify a set of *test vectors* &mdash; suitably chosen inputs with known good outputs &mdash; feeding the inputs from the test vectors to the function under tests and observing the output of the function in response to this stimulus.
Note, however, that for most functions of interest constructing an exhaustive set of test vectors to test functions in this way is infeasible: there are simply too many different vectors that would need to be constructed.

Using CBMC, we can take this idea of 
uses *nondeterministic values* that can be used to model interaction with the outside world or can be used to check that a property holds for arbitrary values of a given type.
To begin, we'll declare (but not define) the following function:

```c
int nondet_int(void);
```

Functions whose name begins with the `nondet_` prefix in CBMC are treated specially by the tool, and are use to generate the aforementioned nondeterministic values &mdash; in this case, an `int`.
Nondeterministically chosen integers are completely arbitrary: there are no constaints on their value, other than those that we choose to impose on them. 

```c
#define ASSERT(cond, name) { __CPROVER_assert(cond, name); }
```

```c
void property0() {
  int arbitrary = nondet_int();

  ASSERT(arbitrary == 0, "property 0");
}
```

```c
void property1() {
  int arbitrary = nondet_int();

  ASSERT(arbitrary != 0, "property 1");
} 
```

```c
void property2() {
  int arbitrary = nondet_int();

  ASSUME(arbitrary == 0);

  ASSERT(arbitrary == 0, "property 2");
}
```


