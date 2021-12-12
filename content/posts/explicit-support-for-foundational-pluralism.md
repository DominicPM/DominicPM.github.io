---
title: "Explicit support for foundational pluralism"
date: 2021-11-27T19:20:37Z
draft: false
tags: ["logic", "verification"]
---

Nearly all working mathematicians make extensive use of non-constructive, classical reasoning principles including the *Law of the Excluded Middle* and the *Axiom of Choice* in their day-to-day work.
Interestingly, despite mostly working classically, mathematicians are able to "drop down" into restricted *modes* of reasoning &mdash; a purely-constructive mode, or a mode where Excluded Middle is permissible but the Axiom of Choice is not, for example &mdash; where and when the need arises.
This foundational *context switch* is often marked in pen-and-paper mathematics with an informal notice in the written narrative of a paper or textbook: "the following proof is constructive", "the following proof does not use the Axiom of Choice", or similar (see [here](https://tcsmath.wordpress.com/2015/11/18/a-geometric-proof-of-okamura-seymour/), for one example, where the proof of Lemma 2 is marked as being constructive in the English-language narrative &mdash; though many other examples can be found easily by Googling that phrase).
These informal notices are left to the reader to verify, when checking the proof.

Surprisingly, despite the ubiquity of these mixed modes of reasoning in pen-and-paper mathematics, proof assistants provide little *ergonomic* support for selectively restricting or enhancing the permissible modes of reasoning a user may use when constructing a proof.
The situation is especially challenging for proof assistants based on classical logic &mdash; the majority of which are implementations of a polymorphic variant of Church's [Simple Type Theory](https://plato.stanford.edu/entries/type-theory-church/), called HOL, such as [Isabelle/HOL](https://isabelle.in.tum.de/), [HOL4](https://hol-theorem-prover.org/), and [HOL Light](https://www.cl.cam.ac.uk/~jrh13/hol-light/) &mdash; which in many cases actively conspire against the user when trying to work *subclassically*.
In short, the problem with HOL is that it is almost *too* strident in its commitment to classical mathematics.

In particular, choice in systems that implement HOL is not introduced as an explicit axiom, as one may typically expect, but is introduced as a polymorphically-typed [Hilbert-style *choice operator*](https://en.wikipedia.org/wiki/Epsilon_calculus), $\epsilon$,  with type $(\alpha \rightarrow Bool) \rightarrow \alpha$ coupled with an axiom $\vdash \exists{x}. P x \longrightarrow P\ (\epsilon\ P)$, or some analogue, to give this operator meaning.
With this, the term $\epsilon\ P$ represents an arbitrary element of type $ɑ$ satisfying the predicate $P$, if any such element exists, and is otherwise undefined.
The advantage of introducing choice as an operator, rather than an axiom, is the ability to use $\epsilon$ in definitions to judiciously choose some element of a particular type: for example, one may define the infimum of a set of partially ordered elements, or the inverse of a bijection, using $\epsilon$.
Note that introducing choice as an operator in this way still allows you to deduce the usual "Axiom" of Choice as a derived theorem (see the material [here](https://isabelle.in.tum.de/library/HOL/HOL/Hilbert_Choice.html), for example).

Naturally, once choice is introduced, we've essentially "crossed the Rubicon" in a few different ways.
First, choice in HOL immediately implies the Law of the Excluded Middle via a variant of [Diaconescu's theorem](https://en.wikipedia.org/wiki/Diaconescu%27s_theorem).
As a result, once choice is introduced we're essentially fully commited to a wholly classical foundation, and once you've made that commitment, you may as well make use of it.
*Everywhere*.
To wit, there's no real way to stop most of Isabelle/HOL's automation and proof-search facilities using classical reasoning when attempting to close an open goal in pursuit of a proof.
Indeed, there's no way of discovering whether the proof of an Isabelle/HOL theorem depends, in some essential way, on choice or excluded middle, though it's probably safe to assume that most do &mdash; even very basic concepts in the Isabelle/HOL standard library, such as the if-then-else eliminator for $Bool$, are defined using a choice operator.

*Note that I'm focussing on Isabelle/HOL here as it's the system with which I am most familiar &mdash; there's nothing specific to Isabelle/HOL in what I'm saying, and I think these observations hold for HOL implementations in general*.

But, is this really a problem in practice?
After all, the Isabelle/HOL standard library and the associated [Archive of Formal Proofs](https://isa-afp.org) shows that a wide range of mathematical content can be formalized in HOL, so what are we really losing out on, concretely?
One *interesting* bit of mathematics that is hard to formalize in Isabelle/HOL, as is, is [Smooth Infinitesimal Analysis](https://en.wikipedia.org/wiki/Smooth_infinitesimal_analysis) (or SIA, henceforth) &mdash; indeed, it was thinking about how SIA can be formalized in Isabelle/HOL which was the genesis for what I will present below.

Very briefly, SIA is an alternative approach to Cohen's [Nonstandard Analysis](https://en.wikipedia.org/wiki/Nonstandard_analysis) in developing analysis using infinitesimals in a fully-rigorous way.
When developing SIA, we fix a set with the suggestive name $\mathbb{R}$ which is taken to be the carrier set of a field.
With this, we define the set of **nipotents**, $\Delta ≝ \\{x \in \mathbb{R} | x \cdot x = 0 \\}$, which are elements of the set $\mathbb{R}$ so imperceptibly close to $0$ that they are equal to $0$ when squared.
Note that *classically* the definition of $\Delta$ collapses, and we can immediately conclude that $\Delta$ is equal to the singleton set, $\\{0\\}$.
Interestingly, however, if we restrict ourselves to using only intuitionistic reasoning, this collapse no longer happens &mdash; instead, all we can say about the set $\Delta$ is that $0 \in \Delta$ and for any $x \in \Delta$ we have $\neg(x \neq 0)$, the latter of which is intuitionistically strictly weaker than the claim $x = 0$.
As a result, we can take the set of nilpotents as our set of infinitesimals and, along with some axioms constraining the behaviour of arbitrary functions $f : \mathbb{R} \rightarrow \mathbb{R}$, develop infintesimal analysis.

Yet, as hinted above, if trying to formalize SIA in Isabelle/HOL, we are immediately hit with a problem: there's no real way of stopping everything collapsing into a triviality.
Instead, it seems the best that we can do is to first embed intuitionistic logic into HOL and then develop SIA inside that embedding &mdash; a kind of proof-assistant within a proof-assistant, which is clearly unsatisfatory.

So, what can be done?

Essentially, in what follows, I'm going to introduce a logic that looks a lot like HOL &mdash; taking that logic as a base &mdash; but is modified in a few minor but important ways.
The most important change will be the introduction of *taint-labels*, which I will use to track the *foundational* axioms used in a proof.
Interestingly, these taint-labels have an internal lattice-like structure, which captures implications between foundational axioms that introduce them into proofs.
Even more interestingly, as discussed further below, these taint-labels appear to be an analogue of the classification labels of [security type systems](https://en.wikipedia.org/wiki/Security_type_system) for programming languages.

### Kinds, types, and terms

First, I'll define **kinds** by the following recursive grammar, where $\star$ is the kind of types:

$
\begin{gathered}
\kappa, \kappa', \kappa'' ::= \star \mid \star \rightarrow \kappa
\end{gathered}
$

The **type** language of the logic remains similar to the polymorphic simple-types of HOL, and is described by the following recursive grammar:

$
\begin{gathered}
\tau, \tau', \tau'' ::= \alpha \mid T_\kappa \mid \tau'\tau
\end{gathered}
$

Here, $\alpha$ ranges arbitrarily over a countably infinite set of **type-variables**, and application $\tau'\tau$ associates to the left.
Moreover, to each kind $\kappa$, I assume a countably infinite set of **type-formers** of that kind.
Note that kinds are isomorphic to the natural numbers, and will be primarily used to describe the **arity** of a type-former, that is, how many type arguments it must be applied to in order to create a type.
If $\kappa \neq \kappa'$ then there is no particular connection between $T_\kappa$ and $T_{\kappa'}$ &mdash; though I will try to avoid confusing name clashes, throughout.

Henceforth, I will only ever work with well-formed types, where every type-former is maximally-applied.
To capture this, I introduce a **type kinding** relation, $\tau : \kappa$, with the following rules:

$
\begin{gathered}
\cfrac{}{\alpha : \star}
\quad
\cfrac{}{T_\kappa : \kappa}
\quad
\cfrac{\tau : \star \rightarrow \kappa \quad \tau : \star}{\tau'\tau : \kappa}
\end{gathered}
$

I will write $\tau : \kappa$ to assert that a derivation tree, rooted at $\tau : \kappa$, and constructed according to the rules above, exists.
Note that this relation satisfies a number of properties:

**Lemma (unicity of kinding)**: If $\tau : \kappa$ and $\tau : \kappa'$ then $\kappa = \kappa'$.

Additionally, I will write $ftv(\tau)$ to denote the **set of type-variables** of the term $\tau$, and write $\tau[\alpha ::= \tau']$ to denote the **substitution** of all occurrences of $\alpha$ for $\tau'$ in the type $\tau$.
Then:

**Lemma**: If $\tau : \kappa$ and $\tau' : \star$ then $\tau[\alpha := \tau'] : \kappa$.

**Lemma**: $ftv(\tau[\beta ::= \tau']) \subseteq (ftv(\tau) - \\{\beta\\}) \cup ftv(\tau')$.

I will need to assume two type-formers as *primitive*, built into the logic itself, in order to develop the rest of the material below:
- The **propositional type**, $Prop$, at kind $\star$,
- The **function space arrow**, $\rightarrow$, at kind $\star \rightarrow \star \rightarrow \star$.

Note that, for ease of reading, instead of writing $(\rightarrow \tau)\tau'$, I will use the standard mathematical convention and write $\tau \rightarrow \tau'$.
Moreover, the function space arrow will be assumed to associate to the right, so we can right $\tau \rightarrow \tau' \rightarrow \tau''$ instead of $\tau \rightarrow (\tau' \rightarrow \tau'')$.
From this, we obviously therefore have the following two derived kinding rules:

$
\begin{gathered}
\cfrac{}{Prop : \star}
\quad
\cfrac{\tau : \star \quad \tau' : \star}{\tau \rightarrow \tau' : \star}
\end{gathered}
$

Observe here that &mdash; compared to HOL &mdash; there is one minor but symbolic change: the distinguished Boolean type of HOL is now renamed to to $Prop$.

In HOL, the $Bool$ type serves a dual purpose, in both identifying formulae and acting as the two-element datatype.
Henceforth &mdash; at least in some circumstances &mdash; these two roles are going to be separated: $Prop$ will be used to identify formulae, whilst $Bool$ can be introduced later as a two-element datatype, like any other.
(I will come back to the subject of data.)
In this way, formulae of this new logic are identified with well-typed terms that inhabit the $Prop$ type, and we will use Greek letters like $\phi$ and $\psi$ to suggest that terms are formulae.

To each type, $\tau$, I assume a countably infinite set of **variables** and **constants**.
I will write $x_{\tau}$ to refer to the variable $x$ associated with type $\tau$, and write $C_\tau$ for the constant similarly associated with type $\tau$.
Again, whenever $\tau \neq \tau'$ there is no particular connection between $x_\tau$ and $x_{\tau'}$, and between $C_\tau$ and $C_{\tau'}$, and I will generally work to avoid introducing confusing name clashes in this way.

Like HOL, the **term language** of the logic remains the terms of the simply-typed $\lambda$-calculus, extended with constants.
Terms are therefore defined by the following recursive grammar:

$
\begin{gathered}
r,s,t ::= x_\tau \mid C_\tau \mid rs \mid \lambda{x_\tau}.r
\end{gathered}
$

I work with terms identified upto $\alpha$-equivalence &mdash; that is, a permutative renaming of their bound variables.
Additionally, it will also generally be the case that I only work with well-typed terms.
In pursuit of this, I introduce a **term typing** relation, defined by the following rules:

$
\begin{gathered}
\cfrac{}{x_\tau : \tau}
\quad
\cfrac{}{C_\tau : \tau}
\quad
\cfrac{r : \tau \rightarrow \tau' \quad s : \tau}{rs : \tau'}
\quad
\cfrac{r : \tau'}{\lambda{x_\tau}.r : \tau \rightarrow \tau'}
\end{gathered}
$

I will write $r : \tau$ to assert that a derivation tree, rooted at $r : \tau$, and constructed using the rules above, exists.
Note that this relation satisfies a number of properties:

**Lemma (unicity of typing)**: if $r : \tau$ and $r : \tau'$ then $\tau = \tau'$.

**Lemma**: if $r : \tau$ then $\tau : \star$.

Moreover, I will write $fv(r)$ to denote the set of **free variables** of the term $r$, $ftv(r)$ to denote the set of **type-variables** appearing in the term $r$, $r[x_\tau \mapsto t]$ for the **capture-avoiding substitution** which replaces all free-occurrences of $x_\tau$ for $t$ in $r$, and lastly writing $r[\alpha \mapsto \tau]$ for the obvious extension of the **type-substitution** to terms.
We then have the following standard properties of these operations:

**Lemma**: $fv(r[x_\tau \mapsto t]) \subseteq (fv(r) - \\{x_\tau\\}) \cup fv(t)$

**Lemma**: $ftv(r[\alpha \mapsto \tau]) \subseteq (ftv(r) - \\{\alpha\\}) \cup ftv(\tau)$

**Lemma**: $fv(r[\alpha \mapsto \tau]) = fv(r)$

**Lemma (Barendregt's substitution lemma)**: if $x_\tau \notin fv(s)$ and $x_\tau \neq y_{\tau'}$ then $r[x_\tau \mapsto t][y_{\tau'} \mapsto s] = r[y_{\tau'} \mapsto s][x_\tau \mapsto t[y_{\tau'} \mapsto s]]$.

**Lemma**: if $x_\tau \notin fv(r)$ then $r[x_\tau \mapsto t] = r$.

**Lemma**: if $\alpha \notin ftv(r)$ then $r[\alpha \mapsto \tau] = r$.

**Lemma**: $r[\alpha \mapsto \alpha] = r$ and $r[x_\tau \mapsto x_\tau] = r$.

**Lemma**: $r[x_\tau \mapsto t][\beta \mapsto \tau'] = r[\beta \mapsto \tau'][x_{\tau[\beta \mapsto \tau']} \mapsto t[\beta \mapsto \tau']]$

**Lemma**: if $r : \tau$ then $r[\beta \mapsto \tau'] : \tau[\beta \mapsto \tau']$.

**Lemma**: if $r : \tau$ and $s : \tau'$ then $r[y_{\tau'} \mapsto s] : \tau$.

Throughout, I will assume the standard gamut of logical connectives and quantifiers, captured as appropriately-typed constants:

- Truth and falsity, $\top$ and $\bot$, at type $Prop$,
- Logical negation, $\neg$, at type $Prop \rightarrow Prop$,
- Conjunction, disjunction, and implication, $\wedge$, $\vee$ and $\longrightarrow$, at type $Prop \rightarrow Prop \rightarrow Prop$,
- Equality, $=$, at type $\alpha \rightarrow \alpha \rightarrow Prop$,
- Existential and universl quantification, $\exists$ and $\forall$, at type $(\alpha \rightarrow Prop) \rightarrow Prop$.

(Note the conspicuous lack of any Hilbert-style choice operator, like the previously-discussed $\epsilon$, another subject which I'll come back to.)

I'll also adopt the usual mathematical typographical conventions when rendering connectives, for example writing $r = s$ and $\exists{x_{\tau}}. \phi$ instead of $(= r)s$ and $\exists$ ($\lambda{x_{\tau}}. \phi$), respectively.
Note also that I will tend to suppress the explicit type annotations of the constants, especially for the polymorphic constants, preferring to simply write $\forall$ instead of $\forall[\alpha \mapsto \tau]$, and similar.

### Natural Deduction

Call a finite set of terms a **context**, and call a context **well-formed**, and write $\Gamma \text{ wf}$, whenever $\phi : Prop$ for every $\phi \in \Gamma$.
I will use $\Gamma$, $\Gamma'$, $\Gamma''$, and so on, to range arbitrarily over contexts.
Writing $\\{\\}$ for the empty context, we have:

**Lemma**: $\\{\\} \text{ wf}$

**Lemma**: if $\Gamma \text{ wf}$ and $\Gamma' \text{ wf}$ then $(\Gamma \cup \Gamma') \text{ wf}$.

**Lemma**: if $\Gamma \text{ wf}$ and $\Gamma' \subseteq \Gamma$ then $\Gamma' \text{ wf}$.

Moreover, I will write $\Gamma[\alpha \mapsto \tau]$ and $\Gamma[x_\tau \mapsto t]$ for the **pointwise extension** of the **type-substitution** and **capture-avoiding substitution** actions to contexts.
In light of this, note that:

**Lemma**: if $\Gamma \text{ wf}$ and $\tau : \star$ then $\Gamma[\alpha \mapsto \tau] \text{ wf}$.

**Lemma**: if $\Gamma \text{ wf}$ and $t : \tau$ then $\Gamma[x_\tau \mapsto t] \text{ wf}$.

To enforce a separation between the intuitionistic and classical *worlds*, I will take a familiar two-place Natural Deduction relation &mdash; $\Gamma$ $\vdash$ $\phi$, between a context of assumptions and a formula &mdash; and extend it to a *three*-place relation, $\Gamma$ $\vdash$ $\phi$ : $\ell$.
Here, $\ell$ ranges arbitrarily over a set of **taint-labels**, $\mathcal{L}$.
For the time being I will assume the existence of two distinguished and suggestively-named labels: $\mathcal{C}$ and $\mathcal{I}$.
Note that more labels will be introduced later.

Essentially, the idea is that, when constructing a proof, taint-labels are introduced by instances of any axioms used and are consequently "bubbled up" through the proof via the logic's inference rules.
To accommodate this, inference rules associated with a connective must be modified slightly to properly handle taint-labels.
This step will require some additional machinery, which will be introduced below.
At this point, however, I will introduce the axioms of the Natural Deduction relation.
We have:

The **Start** axiom, which allows us to work from any assumption appearing in our context, $\Gamma$:

$
\begin{gathered}
\cfrac{(\Gamma \text{ wf} \quad \phi \in \Gamma)}{\Gamma \vdash \phi : \mathcal{I}}
\end{gathered}
$

The **Truth Introduction** axiom, which allows us to unconditionally prove $\top$, the truth constant.
Note that the axiom introduces the $\mathcal{I}$ label, as truth is intuitionistically provable:

$
\begin{gathered}
\cfrac{(\Gamma \text{ wf})}{\Gamma \vdash \top : \mathcal{I}}
\end{gathered}
$

The **Excluded Middle** axiom.
Note that the axiom introduces the $\mathcal{C}$ label, as its use means we are working classically:

$
\begin{gathered}
\cfrac{(\Gamma \text{ wf})}{\Gamma \vdash \phi \vee \neg\phi : \mathcal{C}}
\end{gathered}
$

The **Reflexivity** axiom, which allows us to unconditionally prove that any term is equal to itself.
Note that this axiom also introduces the $\mathcal{I}$ label, as this is also provable, intuitionistically:

$
\begin{gathered}
\cfrac{(\Gamma \text{ wf} \quad r : \tau)}{\Gamma \vdash r = r : \mathcal{I}}
\end{gathered}
$

The **Beta** axiom, which allows us to apply a function to an argument, substituting the argument into the body of the function.
Again, this axiom introduces the $\mathcal{I}$ label:

$
\begin{gathered}
\cfrac{(\Gamma \text{ wf} \quad (\lambda{x_{\tau}}. r)s : \tau')}{\Gamma \vdash (\lambda{x_{\tau}}. r)s = r[x_{\tau} \mapsto s] : \mathcal{I}}
\end{gathered}
$

The **Eta** axiom, which introduces *extensionality* into the logic.
Again, this axiom introduces the $\mathcal{I}$ label:

$
\begin{gathered}
\cfrac{(\Gamma \text{ wf} \quad f : \tau \rightarrow \tau' \quad x_{\tau} \notin fv(f))}{\Gamma \vdash (\lambda{x_{\tau}}. f x) = f : \mathcal{I}}
\end{gathered}
$

Next, to define the inference rules, I first fix a binary operation, $- \sqcup -$, on the set of taint-labels, $\mathcal{L}$, and define an **equational theory** over elements of this set, $\ell$ $\equiv$ $\ell'$:

$
\begin{gathered}
\cfrac{}{\ell \equiv \ell}
\quad
\cfrac{}{\ell \sqcup \ell \equiv \ell}
\quad
\cfrac{\ell \equiv \ell'}{\ell' \equiv \ell}
\quad
\cfrac{\ell \equiv \ell' \quad \ell' \equiv \ell''}{\ell \equiv \ell''}
\newline
\cfrac{\ell_1 \equiv \ell_2 \quad \ell'_1 \equiv \ell'_2}{\ell_1 \sqcup \ell'_1 \equiv \ell_2 \sqcup \ell'_2}
\quad
\cfrac{}{\mathcal{C} \sqcup \ell \equiv \mathcal{C}}
\quad
\cfrac{}{\mathcal{I} \sqcup \ell \equiv \ell}
\end{gathered}
$

I will write $\ell \equiv \ell'$ to assert that a derivation tree exists, rooted at $\ell \equiv \ell'$, and constructed according to the rules above.

Note that this equational theory endows taint-labels with a **join semilattice structure**, under the binary operation $- \sqcup -$, with upper bound $\mathcal{C}$ and lower bound $\mathcal{I}$.
With this, I define a derived ordering relation on taint-labels, and write $\ell \leq \ell'$ to assert that $\ell \sqcup \ell' \equiv \ell'$.
Immediately, we can see that this derived ordering is a **partial ordering** on taint-labels, again with least element $\mathcal{I}$ and greatest element $\mathcal{C}$.

With this, the standard introduction and elimination rules are modified to gather the taint-labels appearing in their premisses and label their conclusion with their least upper bound.
Writing $fv(\Gamma)$ for the set $\bigcup\\{ fv(\phi) \mid \phi \in \Gamma \\}$, we have the following rules:

The **Symmetry** and **Transitivity** rules which capture the symmetric and transitive nature of equality, respectively.
Note that the taint-labels of the premisses are either passed forward, unmodified, in rules with a single premiss, or otherwise combined together, using the $- \sqcup -$ operation on taint-labels, in the conclusion of multi-premiss rules.
Essentially, this captures the fact that, if any premiss is established using some classical reasoning principle, the resulting proof will also be considered suitably classical, too:

$
\cfrac{\Gamma \vdash r = s : \ell}{\Gamma \vdash s = r : \ell}
\quad
\cfrac{\Gamma \vdash r = s : \ell \quad \Delta \vdash s = t : \ell'}{\Gamma ∪ \Delta \vdash r = t : \ell \sqcup \ell'}
$



The **Type Substitution** and **Substitution** rules, which allows us to monomorphise a polymorphic proof, and specialise the free variables of a proof, respectively:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi : \ell}{\Gamma[\alpha \mapsto \tau] \vdash \phi[\alpha \mapsto \tau] : \ell}
\quad
\cfrac{\Gamma \vdash \phi : \ell}{\Gamma[x_{\tau} \mapsto s] \vdash \phi[x_{\tau} \mapsto s] : \ell}
\end{gathered}
$

The **Lambda Congruence** rule which allows us to replace the bodies of functions with a provably-equal term:

$
\cfrac{\Gamma \vdash r = s : \ell \quad (x_\tau \notin fv(\Gamma))}{\Gamma \vdash (\lambda{x_{\tau}}. r) = (\lambda{x_{\tau}}. s) : \ell}
$

The **Application Congruence** rule, which allows us to replace functions and arguments with provably-equal terms.
Note that the side-condition ensures that the resulting applications are well-typed:

$
\cfrac{\Gamma \vdash f = g : \ell \quad \Gamma' \vdash r = s : \ell' \quad (f : \tau \rightarrow \tau', r : \tau)}{\Gamma \cup \Gamma' \vdash f(r) = g(s) : \ell \sqcup \ell'}
$

The **Falsity Elimination** rule, which allows us to conclude *any* well-typed formula from a proof of $\bot$:

$
\cfrac{\Gamma \vdash \bot : \ell \quad (\phi : Prop)}{\Gamma \vdash \phi : \ell}
$

The **Conjunction Introduction** and **Left** and **Right Elimination** rules:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi : \ell \quad \Gamma' \vdash \psi : \ell'}{\Gamma ∪ \Gamma' \vdash \phi \wedge \psi : \ell \sqcup $\ell'$}
\quad
\cfrac{\Gamma \vdash \phi \wedge \psi : \ell}{\Gamma \vdash \phi : \ell}
\quad
\cfrac{\Gamma \vdash \phi \wedge \psi : \ell}{\Gamma \vdash \psi : \ell}
\end{gathered}.
$

The **Implication Introduction** and **Elimination** rules:

$
\begin{gathered}
\cfrac{\Gamma ∪ \\{\phi\\} \vdash \psi : \ell}{\Gamma \vdash \phi \longrightarrow \psi : \ell}
\quad
\cfrac{\Gamma \vdash \phi \longrightarrow \psi : \ell \quad \Gamma' \vdash \phi : \ell'}{\Gamma \cup \Gamma' \vdash \psi : \ell \sqcup \ell'}
\end{gathered}
$


The **Negation Introduction** and **Elimination** rules:

$
\begin{gathered}
\cfrac{\Gamma \cup \\{\phi\\} \vdash \bot : \ell}{\Gamma \vdash \neg\phi : \ell}
\quad
\cfrac{\Gamma \vdash \phi : \ell \quad \Gamma' \vdash \neg\phi : \ell'}{\Gamma \cup \Gamma' \vdash \bot : \ell \sqcup \ell'}
\end{gathered}
$

The **Bi-implication Introduction** and **Left** and **Right Elimination** rules (note that bi-implication is equality at type $Prop$):

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi \longrightarrow \psi : \ell \quad \Gamma' \vdash \psi \longrightarrow \phi : \ell'}{\Gamma \cup \Gamma' \vdash \phi = \psi : \ell \sqcup \ell'}
\quad
\cfrac{\Gamma \vdash \phi = \psi : \ell}{\Gamma \vdash \phi \longrightarrow \psi : \ell}
\\\\
\cfrac{\Gamma \vdash \phi = \psi : \ell}{\Gamma \vdash \psi \longrightarrow \phi : \ell}
\end{gathered}
$

The **Left** and **Right Disjunction Introduction** and **Elimination** rules:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi : \ell}{\Gamma \vdash \phi \vee \psi : \ell}
\quad
\cfrac{\Gamma \vdash \psi : \ell}{\Gamma \vdash \phi \vee \psi : \ell}
\\\\
\cfrac{\Gamma \vdash \phi \vee \psi : \ell \quad \Gamma \cup \\{\phi\\} \vdash \xi : \ell' \quad \Gamma \cup \\{\psi\\} \vdash \xi : \ell''}{\Gamma \vdash \xi : \ell \sqcup (\ell' \sqcup \ell'')}
\end{gathered}
$

The **Universal Introduction** and **Elimination** rules:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi : \ell \quad (x_\tau \notin fv(\Gamma))}{\Gamma \vdash \forall{x_\tau}.\phi : \ell}
\quad
\cfrac{\Gamma \vdash \forall{x_\tau}.\phi : \ell \quad (t : \tau)}{\Gamma \vdash \phi[x_\tau \mapsto t] : \ell}
\end{gathered}
$

The **Existential Introduction** and **Elimination** rules:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi[x_{\tau} \mapsto t] : \ell}{\Gamma \vdash \exists{x_{\tau}}. \phi : \ell}
\quad
\cfrac{\Gamma \vdash \exists{x_{\tau}}. \phi : \ell \quad \Gamma \cup {\phi[x_{\tau} \mapsto y_{\tau}]} \vdash \psi : \ell' \quad (y_{\tau} \notin fv(\Gamma))}{\Gamma \vdash \psi : \ell \sqcup \ell'}
\end{gathered}
$

Note that many of these inference rules are redundant, and could be derived, admittedly.
Henceforth, I will write $\Gamma \vdash \phi : \ell$ to assert that a derivation tree exists, rooted at $\Gamma \vdash \phi : \ell$, and constructed using the rules described above, exists.

If $\\{\\} \vdash \phi : \ell$ then call $\phi$ a **theorem**.
At this point, we now have two separate "worlds" within which a theorem may live: $\mathcal{I}$, rather obviously, carves out the purely-intuitionistic fragment of the logic, whilst $\mathcal{C}$, again rather obviously, demarcates the classical fragment.
Note that there are two ways of interpreting the taint-labels, depending on whether we are working *forwards* from axioms toward conclusions, or *backwards* from conclusions toward axioms, in a proof:

- For forwards proof, taint-labels are used to indelibly mark the proof of a theorem whenever a classical reasoning principle is used, and moreover indelibly mark any other result that makes use of that proof. 
- For backwards proof, taint-labels are used to create a limit on the types of reasoning principles that may be used when working backward.
In the context of our taint-label semilattice, working backwards from $\Gamma \vdash \phi \wedge \psi : \mathcal{I}$ essentially constrains us to proving $\Gamma \vdash \phi : \mathcal{I}$ and $\Gamma \vdash \psi : \mathcal{I}$, as no other taint-label sits below $\mathcal{I}$ in the semilattice.

However, thus far, the two worlds within the logic are a little *too* separate, as we have no way of making a purely-intuitionistic result usable from the classical fragment of our logic.
To handle this, I must introduce a *new* inference rule, the **Embedding** rule:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi : \ell \quad (\ell \leq \ell')}{\Gamma \vdash \phi : \ell'}
\end{gathered}
$

The **Embedding** rule, above, serves two purposes: first, it allows us to *lift* an intuitionistic result into the classical fragment of the logic, and secondly it also has the effect of ensuring that the Natural Deduction relation remains well-defined with respect to the equational theory on taint-labels.
Note that this latter aspect is a direct corollary of the fact that $\ell \equiv \ell'$ implies $\ell \leq \ell'$.
This idea can be expressed as a slightly-modified, derived form of the **Embedding** rule &mdash; called the **Well-Definedness** rule &mdash; as follows:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi : \ell \quad (\ell \equiv \ell')}{\Gamma \vdash \phi : \ell'}
\end{gathered}
$

With this, the bracketing of taint-labels in the conclusion of the **Disjunction Elimination** rule, for example, is largely irrelevant, as the binary operation, $- \sqcup -$, is associative and can be bracketed arbitrarily with no change in meaning. 

The Natural Deduction relation satisfies a number of basic correctness properties.
First, only well-formed contexts and formulae are in-relation:

**Lemma**: if $\Gamma \vdash \phi : \ell$ then $\Gamma \text{ wf}$.

**Lemma**: if $\Gamma \vdash \phi : \ell$ then $\phi : Prop$.

Moreover, if one can derive an equality between terms, via the Natural Deduction relation, then the equated terms have the same type (and that type is itself well-formed):

**Lemma**: if $\Gamma \vdash r = s : \ell$ then $r : \phi$ and $s : \phi$ for some $\phi : \star$.

At this point, we can introduce a few more derived and admissible rules which are of general utility when constructing a proof.
Note that neither of these rules will shock anybody with any familiarity with basic proof theory, albeit they now also need to be suitably modified to make them aware of the taint-labels.

First, the **Weakening** rule, which is proved by induction on the derivation of $\Gamma \vdash \phi : \ell$, allows us to add arbitrarily many further assumptions to the context that we are working in:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi : \ell \quad (\Gamma \subseteq \Gamma' \quad \Gamma' \text{ wf})}{\Gamma' \vdash \phi : \ell}
\end{gathered}
$

As a corollary of the **Weakening** rule, we can also introduce a form of **Cut** which allows us to introduce a useful lemma over the course of the proof:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi : \ell \quad \Gamma' \cup \\{ \phi \\} \vdash \psi : \ell'}{\Gamma \cup \Gamma' \vdash \psi : \ell \sqcup \ell'}
\end{gathered}
$

### Some more derived rules

Here, we start to explore other useful and interesting derived rules.
We have various **generalised congruence** rules:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi_1 = \phi_2 : \ell \quad \Gamma' \vdash \psi_1 = \psi_2 : \ell'}{\Gamma \cup \Gamma' \vdash (\phi_1 \wedge \psi_1) = (\phi_2 \wedge \psi_2) : \ell \sqcup \ell'}
\quad
\cfrac{\Gamma \vdash \phi_1 = \phi_2 : \ell \quad \Gamma' \vdash \psi_1 = \psi_2 : \ell'}{\Gamma \cup \Gamma' \vdash (\phi_1 \vee \psi_1) = (\phi_2 \vee \psi_2) : \ell \sqcup \ell'}
\newline
\cfrac{\Gamma \vdash \phi_1 = \phi_2 : \ell \quad \Gamma' \vdash \psi_1 = \psi_2 : \ell'}{\Gamma \cup \Gamma' \vdash (\phi_1 \longrightarrow \psi_1) = (\phi_2 \longrightarrow \psi_2) : \ell \sqcup \ell'}
\quad
\cfrac{\Gamma \vdash \phi_1 = \phi_2 : \ell}{\Gamma \vdash \neg\phi_1 = \neg\phi_2 : \ell}
\newline
\cfrac{\Gamma \vdash \phi_1 = \phi_2 : \ell \quad (x_\tau \notin fv(\Gamma))}{\Gamma \vdash (\forall{x_\tau}.\phi_1) = (\forall{x_\tau}.\phi_2) : \ell}
\quad
\cfrac{\Gamma \vdash \phi_1 = \phi_2 : \ell \quad (x_\tau \notin fv(\Gamma))}{\Gamma \vdash (\exists{x_\tau}.\phi_1) = (\exists{x_\tau}.\phi_2) : \ell}
\end{gathered}
$

An **extensional equality** rule for functions, capturing the fact that functions are equal when they behave the same on all possible inputs:

$
\begin{gathered}
\cfrac{\Gamma \vdash f(x_{\tau}) = g(x_{\tau}) : \ell \quad (x_{\tau} \notin fv(\Gamma))}{\Gamma \vdash f = g : \ell}
\end{gathered}
$

Moreover, we have various derived **classical reasoning principles**, including:

$
\begin{gathered}
\cfrac{\Gamma \cup \\{\neg\phi\\} \vdash \bot : \ell}{\Gamma \vdash \phi : \mathcal{C}}
\quad
\cfrac{(\Gamma \text{ wf} \quad \phi : Prop \quad \psi : Prop)}{\Gamma \vdash ((\phi \rightarrow \psi) \rightarrow \phi) \rightarrow \phi : \mathcal{C}}
\newline
\cfrac{\Gamma \cup \\{\phi\\} \vdash \psi : \ell \quad \Gamma \cup \\{\neg\phi\\} \vdash \psi : \ell'}{\Gamma \vdash \psi : \mathcal{C}}
\quad
\cfrac{(\Gamma \text{ wf} \quad \phi : Prop \quad \psi : Prop)}{\Gamma \vdash (\phi \longrightarrow \psi) = (\neg\phi \vee \psi) : \mathcal{C}}
\newline
\cfrac{(\Gamma \vdash \text{ wf} \quad \phi : Prop)}{\Gamma \vdash (\exists{x_\tau}.\phi) = (\neg\forall{x_\phi}.(\neg\phi)) : \mathcal{C}}
\quad
\cfrac{\Gamma \vdash \neg\neg\phi : \ell}{\Gamma \vdash \phi : \mathcal{C}}
\end{gathered}
$

...and so on.

### Definitions and data

We some definitional mechanisms to extend our global theory.
I'll start with the simplest mechanism which introduces a new constant with an associated defining theorem.
If fv r = {} and r is typeable with r : $\tau$ then introduce a new constant C : $\tau$ and an additional axiom:

- The *definitional axiom for C*: {} $\vdash$ C = r : $\mathcal{I}$

Note that placing the defining theorem in the intuitionistic fragment of the logic allows both intuitionists and classical mathematicians alike to make use of the definition &mdash; essentially both groups are able to reason about the same objects, albeit they're able to deduce different properties of those objects.
In principle, the defining theorem introduced by a new definition could be placed into the classical fragment of the logic &mdash; that is, labelled with the $\mathcal{C}$ taint-label &mdash; thus rendering its use inaccessible to intuitionists, but it's not immediately clear to me why that would be done (maybe this is a lack of imagination on my part, though).

Next, we need some way of introducing data.
Unlike typical HOL implementations, such as Isabelle/HOL, HOL4, and HOL Light, I'll add a definitional principle which adds strictly-positive datatypes to the theory directly, rather than gradually building up material so that such a definitional facility can be defined.
In particular, for a datatype T with constructors Cᵢ : $\tau$ᵢ for 1 $\leq$ i $\leq$ n where $\tau$ᵢ are strictly-positive in T (and are all injections *into* T) I assume:

- A primitive recursion combinator for T, with defining theorems in $\mathcal{I}$, called "T.rec".
- A structural induction principle for T, in $\mathcal{I}$, called "T.induct".

The recursion combinator will allow us to define operations on elements of T, whilst the induction principle allows us to prove properties of them.
Note the restriction to primitive recursion will be quite limiting in general, but will suffice here for developing examples.
I may sometimes assume Bool and Nat, two strictly-positive datatypes, in what follows.
The first is the usual two-element type, whilst the second is a type representing unary natural numbers, with respective constructors:

- *Constructors* for the Bool type:
  - true : Bool,
  - false : Bool.
- *Constructors* for the Nat type:
  - zero : Nat,
  - succ : Nat $\rightarrow$ Nat.

Moreover, we have the following structural induction principles:

- The *Bool induction* axiom, Bool.induct: {} $\vdash$ $\forall$P:Bool $\rightarrow$ Prop. $\forall$b:Bool. P true $\longrightarrow$ P false $\longrightarrow$ P b : $\mathcal{I}$,
- The *Nat induction* axiom, Nat.induct: {} $\vdash$ $\forall$P:Nat $\rightarrow$ Prop. $\forall$n: Nat. P zero $\longrightarrow$ ($\forall$n:Nat. P n $\longrightarrow$ P (succ n)) $\longrightarrow$  P n : $\mathcal{I}$,

Finally, we have the three primitive recursion combinators for the respective types: Bool.rec, and Nat.rec:
These have the following defining axioms:

- The **definitional axioms** for *Bool.rec*:
  - *{} $\vdash$ $\forall$t:ɑ. $\forall$f:ɑ. Bool.rec true t f = t : $\mathcal{I}$*,
  - *{} $\vdash$ $\forall$t:ɑ. $\forall$f:ɑ. Bool.rec false t f = f : $\mathcal{I}$*
- The **definitional axioms** for *Nat.rec*:
  - *{} $\vdash$ $\forall$z:ɑ. $\forall$s:ɑ $\rightarrow$ ɑ. Nat.rec zero z s = z : $\mathcal{I}$*,
  - *{} $\vdash$ $\forall$m:Nat. $\forall$z:ɑ. $\forall$s:ɑ $\rightarrow$ ɑ. Nat.rec (succ m) z s = s (Nat.rec m z s) : $\mathcal{I}$*

Henceforth, I will sometimes write *if b then t else f* instead of *Bool.rec b t f*. 

### Internal set theory

I'll introduce a new unary type-former, Set.
Intuitively, Set $\tau$ is the type of potentially infinite sets containing elements of type $\tau$.
In addition, I'll introduce two new polymorphially-typed constants, Collect with type (ɑ $\rightarrow$ Prop) $\rightarrow$ Set ɑ and Project with type Set ɑ $\rightarrow$ ɑ $\rightarrow$ Prop.
(Note, in what follows, I will use the two constants more akin to a parameterised family of types, using some sleight-of-hand.)
The two constants are connected by the following two axioms:

- The *Collect-Project* axiom: {} $\vdash$ $\forall$S: Set ɑ. Collect (Project S) = S : $\mathcal{I}$.
- The *Project-Collect* axiom: {} $\vdash$ $\forall$P: ɑ $\rightarrow$ Prop. Project (Collect P) = P : $\mathcal{I}$ 

...which place our sets of elements of a given type, $\tau$, in isomorphism with predicates over $\tau$.

Note that I'll sometimes write { $x_{\tau}$ | $\phi$ } as a shorthand for the term Collect ($\lambda$$x_{\tau}$. $\phi$).
In addition, I'm going to introduce new polymorphically-typed constants for familiar operations on sets, with defining theorems:

- The *Set Membership* definition: {} $\vdash$ $\in$ = $\lambda$x:ɑ. $\lambda$S: Set ɑ. Project S x : $\mathcal{I}$.
- The *Empty Set* definition: {} $\vdash$ ∅ = { x:ɑ | $$\bot$$ } : $\mathcal{I}$.
- The *Universal Set* definition: {} $\vdash$ Universal = { x:ɑ | $$\top$$ } : $\mathcal{I}$
- The *Subset* definition: {} $\vdash$ $\subseteq$ = $\lambda$S: Set ɑ. $\lambda$T: Set ɑ. $\forall$x:ɑ. x $\in$ S $\longrightarrow$ x $\in$ T : $\mathcal{I}$.
- The *Union* definition: {} $\vdash$ ∪ = $\lambda$S: Set ɑ. $\lambda$T: Set ɑ. { x:ɑ | x $\in$ S $\vee$ x $\in$ T } : $\mathcal{I}$.
- The *Intersection* definition: {} $\vdash$ ∩ = $\lambda$S: Set ɑ. $\lambda$T: Set ɑ. { x:ɑ | x $\in$ S $\wedge$ x $\in$ T } : $\mathcal{I}$.
- The *Set Complement* definition: {} $\vdash$ - = $\lambda$S: Set ɑ. { x:ɑ | $\neg$(x:ɑ $\in$ S) } : $\mathcal{I}$.

With this, we have the following derived rules:

- The *Universal Complement* rule: $\Gamma$ $\vdash$ - Universal = ∅ : $\mathcal{I}$.
- The *Double Complement* rule: $\Gamma$ $\vdash$ --S = S : $\mathcal{C}$.
- The *Union-Intersection De Morgan* rule: $\Gamma$ $\vdash$ -(S ∪ T) = -S ∩ -T : $\mathcal{C}$.
- The *Set Equality Introduction* rule: if $\Gamma$ $\vdash$ $\phi$ $\subseteq$ T : $\ell$ and \Delta $\vdash$ T $\subseteq$ S : $\ell'$ then $\Gamma$ ∪ \Delta $\vdash$ S = T : $\ell$ ⊓ $\ell'$.
- The *Set Cases* rule: if $\Gamma$ ∪ {S = ∅} $\vdash$ $\phi$ : $\ell$ and $\Gamma$ ∪ {$\neg$(S = ∅)} $\vdash$ $\phi$ : $\ell'$ then $\Gamma$ $\vdash$ $\phi$ : $\mathcal{C}$.

...and so on.

### Intermediate logics

Earlier, I mentioned the possibility of introducing further taint-labels beyond the $\mathcal{I}$ and $\mathcal{C}$ labels already introduced.
Here, I'll briefly explore one reason for doing this: further refining our rather coarse-grained split of the logic into two "worlds" by considering additional axioms.
These axioms capture superintuitionistic reasoning principles which result in stronger logics than the base intuitionistic logic but which are nevertheless weaker than classical logic.

I'll therefore add a new taint-label, $\mathcal{W}$, and update the equational theory, accordingly to position this new taint-label between $\mathcal{I}$ and $\mathcal{C}$, so that $\mathcal{I}$ $\leq$ $\mathcal{W}$ and $\mathcal{W}$ $\leq$ $\mathcal{C}$.
In addition, I'll add a further axiom to the logic:

- The *Weak Excluded Middle* axiom: $\Gamma$ $\vdash$ $\neg$$\phi$ $\vee$ $\neg$$\neg$$\phi$ : $\mathcal{W}$.

### Choice

### Discussion
