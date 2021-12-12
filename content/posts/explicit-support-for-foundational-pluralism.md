---
title: "Explicit support for foundational pluralism"
date: 2021-11-27T19:20:37Z
draft: true
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
Note that I'm focussing on Isabelle/HOL here as it's the system with which I am most familiar &mdash; there's nothing specific to Isabelle/HOL in what I'm saying, and I think these observations hold for HOL implementations in general.

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

### A higher-order logic with a taint system

So, what can be done?
Essentially, I'm going to introduce a logic that looks a lot like HOL &mdash; taking that logic as a base &mdash; but is modified in a few minor but important ways.

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

Note that this relation satisfies a number of properties, including *unicity* in the sense that $\tau : \kappa$ and $\tau : \kappa'$ implies $\kappa = \kappa'$.
Additionally, I will write $ftv(\tau)$ to denote the set of type-variables of the term $\tau$, and write $\tau[\alpha ::= \tau']$ to denote the **substitution** of all occurrences of $\alpha$ for $\tau'$ in the type $\tau$.
Then, if $\tau : \kappa$ and $\tau' : \star$ then $\tau[\alpha := \tau'] : \kappa$ and moreover $ftv(\tau[\alpha ::= \tau']) \subseteq ftv(\tau) \cup ftv(\tau')$.

I will assume two type-formers as *primitive*, built into the logic itself:
- The propositional type, $Prop$, at kind $\star$,
- The function space arrow, $\rightarrow$, at kind $\star \rightarrow \star \rightarrow \star$.

Note here that, compared to HOL, there is one minor but symbolic change: the distinguished Boolean type of HOL is now renamed to to $Prop$.

In HOL, the $Bool$ type serves a dual purpose, in both identifying formulae and acting as the two-element datatype.
Henceforth &mdash; at least in some circumstances &mdash; these two roles are going to be separated: $Prop$ will be used to identify formulae, whilst $Bool$ can be introduced later as a two-element datatype, like any other.
(I will come back to the subject of data.)
In this way, formulae of this new logic are identified with well-typed terms that inhabit the $Prop$ type, and we will use Greek letters like $\phi$ and $\psi$ to suggest that terms are formulae.

Henceforth, instead of writing $(\rightarrow \tau)\tau'$ I will use standard mathematical conventions, using the infix notation $\tau \rightarrow \tau'$.

Like HOL, the term language of the logic remains the simply-typed $\lambda$-calculus, with constants, defined by the following recursive grammar:

$
\begin{gathered}
r,s,t ::= x_\tau \mid C_\tau \mid rs \mid \lambda{x_\tau}.r
\end{gathered}
$

To each type, $\tau$, I assume a countably infinite set of **variables** and **constants**.
I will write $x_{\tau}$ to refer to the variable $x$ associated with type $\tau$, and write $C_\tau$ for the constant similarly associated with type $\tau$.
Again, whenever $\tau \neq \tau'$ there is no particular connection between $x_\tau$ and $x_{\tau'}$, and $C_\tau$ and $C_{\tau'}$.

I work with terms identified upto $\alpha$-equivalence, that is, a permutative renaming of their bound variables.
Additionally, henceforth, I will generally work with well-typed terms.
In pursuit of this, I introduce a **term typing** relation, with the following rules:

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

Note that if $r : \tau$ then $\tau : \star$, and if $r : \tau$ and $r : \tau'$ then $\tau = \tau'$.

I will assume the standard gamut of logical connectives and quantifiers, captured as appropriately-typed constants (though I will generally suppress the types for readability's sake): $\top$, $\bot$, $\exists$, $\forall$, $=$, $\longrightarrow$, $\neg$, $\wedge$, and $\vee$ &mdash; though, note the conspicuous lack of any Hilbert-style choice operator, like the previously-discussed $\epsilon$, another subject which I'll come back to.
I'll also adopt the usual mathematical typographical conventions when rendering connectives, for example writing $r = s$ and $\exists$$x_{\tau}$. $\phi$ instead of $(= r)s$ and $\exists$ ($\lambda$$x_{\tau}$. $\phi$), respectively. 

Next, to enforce a separation between the intuitionistic and classical *worlds*, I will take a familiar two-place Natural Deduction relation &mdash; $\Gamma$ $\vdash$ $\phi$, between a context of assumptions and a formula &mdash; and extend it to a *three*-place relation, $\Gamma$ $\vdash$ $\phi$ : $\ell$.
Here, $\ell$ ranges arbitrarily over a set of *taint-labels*, $\mathcal{L}$, and for the time being I will assume the existence of two distinguished and suggestively-named labels, $\mathcal{C}$ and $\mathcal{I}$, though more will be introduced later.

Essentially, the idea is that, when constructing a proof, taint-labels are introduced by instances of any axioms used and are consequently "bubbled up" through the proof via the logic's inference rules.
To accommodate this, inference rules associated with a connective must be *enlightened* to properly handle taint-labels.
This step will require some additional machinery, which will be introduced below.
First, though, I will introduce the axioms:

The **Start** axiom, which allows us to work from any assumption appearing in our context, $\Gamma$:

$
\begin{gathered}
\cfrac{(\phi \in \Gamma)}{\Gamma \vdash \phi : \mathcal{I}}
\end{gathered}
$

The **Truth Introduction** axiom, which allows us to unconditionally prove $\top$, the truth constant.
Note that the axiom introduces the $\mathcal{I}$ label, as truth is intuitionistically provable:

$
\begin{gathered}
\cfrac{}{\Gamma \vdash \top : \mathcal{I}}
\end{gathered}
$

The **Excluded Middle** axiom.
Note that the axiom introduces the $\mathcal{C}$ label, as its use means we are working classically:

$
\begin{gathered}
\cfrac{}{\Gamma \vdash \phi \vee \neg\phi : \mathcal{C}}
\end{gathered}
$

The **Reflexivity** axiom, which allows us to unconditionally prove that any term is equal to itself.
Note that this axiom also introduces the $\mathcal{I}$ label, as this is also provable, intuitionistically:

$
\begin{gathered}
\cfrac{}{\Gamma \vdash r = r : \mathcal{I}}
\end{gathered}
$

The **Beta** axiom, which allows us to apply a function to an argument, substituting the argument into the body of the function.
Here, $r[x_\tau \mapsto s]$ denotes the *capture-avoiding substitution* of occurences of $x_\tau$ for $s$ in the term $r$.
Again, this axiom introduces the $\mathcal{I}$ label:

$
\begin{gathered}
\cfrac{}{\Gamma \vdash (\lambda{x_{\tau}}. r)s = r[x_{\tau} \mapsto s] : \mathcal{I}}
\end{gathered}
$

The **Eta** axiom, which introduces *extensionality* into the logic.
Again, this axiom introduces the $\mathcal{I}$ label:

$
\begin{gathered}
\cfrac{(x_{\tau} \notin fv(f))}{\Gamma \vdash (\lambda{x_{\tau}}. f x) = f : \mathcal{I}}
\end{gathered}
$

Next, to define the inference rules, I first assume a binary operation, $- \sqcup -$, on the set $\mathcal{L}$ and define an equational theory over elements of this set, $\ell$ $\equiv$ 𝑚:

$
\begin{gathered}
\cfrac{}{\ell \equiv \ell}
\quad
\cfrac{}{\ell \sqcup \ell \equiv \ell}
\quad
\cfrac{\ell \equiv 𝑚}{𝑚 \equiv \ell}
\quad
\cfrac{\ell \equiv 𝑚 \quad 𝑚 \equiv 𝑛}{\ell \equiv 𝑛}
\quad
\cfrac{\ell_1 \equiv \ell_2 \quad 𝑚_1 \equiv 𝑚_2}{\ell_1 \sqcup 𝑚_1 \equiv \ell_2 \sqcup 𝑚_2}
\newline
\cfrac{}{\mathcal{C} \sqcup \ell \equiv \mathcal{C}}
\quad
\cfrac{}{\mathcal{I} \sqcup \ell \equiv \ell}
\end{gathered}
$

Note that this equational theory endows taint-labels with a **join semilattice structure**, under the binary operation $- \sqcup -$, with upper bound $\mathcal{C}$ and lower bound $\mathcal{I}$.
With this, I define a derived ordering relation on taint-labels, and write $\ell \leq 𝑚$ to assert that $\ell \sqcup 𝑚 \equiv 𝑚$.
Immediately, we can see that this derived ordering is a **partial ordering** on taint-labels, again with least element $\mathcal{I}$ and greatest element $\mathcal{C}$.

With this, the standard introduction and elimination rules are modified to gather the taint-labels appearing in their premisses and label their conclusion with their least upper bound.
We have the following rules:

The **Symmetry** rule which captures the symmetric nature of equality.
Note that the taint-label of the premiss is preserved, passed down into the conclusion of the rule:

$
\cfrac{\Gamma \vdash r = s : \ell}{\Gamma \vdash s = r : \ell}
$

The **Transitivity** rule which captures the transitivity of equality.
Note that the taint-labels of the premisses are combined together, using the $- \sqcup -$ operation on taint-labels, in the conclusion of the rule.
Essentially, this captures the fact that, if either premiss is established using some classical reasoning principle, the resulting proof will also be considered suitably classical, too:

$
\cfrac{\Gamma \vdash r = s : \ell \quad \Delta \vdash s = t : 𝑚}{\Gamma ∪ \Delta \vdash r = t : \ell \sqcup 𝑚}
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
\cfrac{\Gamma \vdash f = g : \ell \quad \Delta \vdash r = s : 𝑚 \quad (f : \tau \rightarrow \tau', r : \tau)}{\Gamma \cup \Delta \vdash f(r) = g(s) : \ell \sqcup 𝑚}
$

The **Falsity Elimination** rule, which allows us to conclude *anything* from a proof of $\bot$, or false:

$
\cfrac{\Gamma \vdash \bot : \ell}{\Gamma \vdash \phi : \ell}
$

The **Conjunction Introduction** and **Left** and **Right Elimination** rules:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi : \ell \quad \Delta \vdash \psi : 𝑚}{\Gamma ∪ \Delta \vdash \phi \wedge \psi : \ell \sqcup 𝑚}
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
\cfrac{\Gamma \vdash \phi \longrightarrow \psi : \ell \quad \Delta \vdash \phi : 𝑚}{\Gamma \cup \Delta \vdash \psi : \ell \sqcup 𝑚}
\end{gathered}
$


The **Negation Introduction** and **Elimination** rules:

$
\begin{gathered}
\cfrac{\Gamma \cup \\{\phi\\} \vdash \bot : \ell}{\Gamma \vdash \neg\phi : \ell}
\quad
\cfrac{\Gamma \vdash \phi : \ell \quad \Delta \vdash \neg\phi : 𝑚}{\Gamma \cup \Delta \vdash \bot : \ell \sqcup 𝑚}
\end{gathered}
$

The **Bi-implication Introduction** and **Left** and **Right Elimination** rules:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi \longrightarrow \psi : \ell \quad \Delta \vdash \psi \longrightarrow \phi : 𝑚}{\Gamma \cup \Delta \vdash \phi = \psi : \ell \sqcup 𝑚}
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
\cfrac{\Gamma \vdash \phi \vee \psi : \ell \quad \Gamma \cup \\{\phi\\} \vdash \xi : 𝑚 \quad \Gamma \cup \\{\psi\\} \vdash \xi : 𝑛}{\Gamma \vdash \xi : \ell \sqcup (𝑚 \sqcup 𝑛)}
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
\cfrac{\Gamma \vdash \exists{x_{\tau}}. \phi : \ell \quad \Gamma \cup {\phi[x_{\tau} \mapsto y_{\tau}]} \vdash \psi : 𝑚 \quad (x_{\tau} \notin fv(\Gamma))}{\Gamma \vdash \psi : \ell \sqcup 𝑚}
\end{gathered}
$

Note that many of these inference rules are redundant, and could be derived, admittedly.

At this point, we now have two separate "worlds" within which a theorem may live: $\mathcal{I}$, rather obviously, carves out the purely-intuitionistic fragment of the logic, whilst $\mathcal{C}$, again rather obviously, demarcates the classical fragment.
Note that there are two ways of interpreting the taint-labels, depending on whether we are working *forwards* from axioms toward conclusions, or *backwards* from conclusions toward axioms, in a proof:

- For forwards proof, taint-labels are used to indelibly mark a proof whenever a classical reasoning principle is used, and moreover indelibly mark any other result that makes use of that proof. 
- For backwards proof, taint-labels are used to create a limit on the types of reasoning principles that may be used when working backward.
In the context of our taint-label semilattice, working backwards from $\Gamma \vdash \phi \wedge \psi : \mathcal{I}$ essentially constrains us to proving $\Gamma \vdash \phi : \mathcal{I}$ and $\Gamma \vdash \psi : \mathcal{I}$, as no other taint-label sits below $\mathcal{I}$ in the semilattice.

However, thus far we have no way of making a purely-intuitionistic result usable from the classical fragment of our logic.
To handle this, I must introduce a *new* inference rule, the **Embedding** rule:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi : \ell \quad (\ell \leq 𝑚)}{\Gamma \vdash \phi : 𝑚}
\end{gathered}
$

The **Embedding** rule, above, serves two purposes: first, it allows us to "lift" an intuitionistic result into the classical fragment of the logic, and secondly it also has the effect of ensuring that the Natural Deduction relation remains well-defined with respect to the equational theory on taint-labels.
Note that this latter aspect is a direct corollary of the fact that $\ell \equiv 𝑚$ implies $\ell \leq 𝑚$.
This can be expressed as a slightly-modified derived form of the **Embedding** inference rule, called the **Well-Definedness** rule, as follows:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi : \ell \quad (\ell \equiv 𝑚)}{\Gamma \vdash \phi : 𝑚}
\end{gathered}
$

With this, the bracketing of taint-labels in the conclusion of the **Disjunction Elimination** rule, for example, is largely irrelevant, as the binary operation, $- \sqcup -$, is associative per our equational theory. 

At this point, we can introduce a few more derived and admissible rules which are of general utility when constructing a proof.
Note that neither of these rules will shock anybody with any familiarity with basic proof theory, albeit they now also need to be suitably "enlightened" to make them aware of the taint-labels.

First, the **Weakening** rule, which is proved by induction on the derivation of $\Gamma \vdash \phi : \ell$, allows us to add arbitrarily many further assumptions to the context that we are working in:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi : \ell \quad (\Gamma \subseteq \Delta)}{\Delta \vdash \phi : \ell}
\end{gathered}
$

As a corollary of the **Weakening** rule, we can also introduce a form of **Cut** which allows us to introduce a useful lemma over the course of the proof:

$
\begin{gathered}
\cfrac{\Gamma \vdash \phi : \ell \quad \Delta \cup \\{ \phi \\} \vdash \psi : 𝑚}{\Gamma \cup \Delta \vdash \psi : \ell \sqcup 𝑚}
\end{gathered}
$

In addition, we can start to explore other useful and interesting derived rules, such as the following:

- The *Conjunction Congruence* rule: if $\Gamma$ $\vdash$ $\phi$₁ = $\phi$₂ : $\ell$ and \Delta $\vdash$ $\psi$₁ = $\psi$₂ : 𝑚 then $\Gamma$ ∪ \Delta $\vdash$ $\phi$₁ $\wedge$ $\psi$₁ = $\phi$₂ $\wedge$ $\psi$₂ : $\ell$ ⊓ 𝑚.
- The *Function Extensionality* rule: if $\Gamma$ $\vdash$ f $x_{\tau}$ = g $x_{\tau}$ : $\ell$ then $\Gamma$ $\vdash$ f = g : $\ell$ whenever $x_{\tau}$ $\notin$ fv $\Gamma$. 
- The *Classical Contradiction* rule: if $\Gamma$ ∪ {$\neg$$\phi$} $\vdash$ $$\bot$$ : $\ell$ then $\Gamma$ $\vdash$ $\phi$ : $\mathcal{C}$.
- The *Propositional Cases* rule: if $\Gamma$ ∪ {$\phi$} $\vdash$ $\psi$ : $\ell$ and $\Gamma$ ∪ {$\neg$$\phi$} $\vdash$ $\psi$ : 𝑚 then $\Gamma$ $\vdash$ $\psi$ : $\mathcal{C}$.
- The *Classical Implication Definition* axiom: $\Gamma$ $\vdash$ ($\phi$ $\longrightarrow$ $\psi$) = ($\neg$$\phi$ $\vee$ $\psi$) : $\mathcal{C}$.

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
- The *Set Equality Introduction* rule: if $\Gamma$ $\vdash$ $\phi$ $\subseteq$ T : $\ell$ and \Delta $\vdash$ T $\subseteq$ S : 𝑚 then $\Gamma$ ∪ \Delta $\vdash$ S = T : $\ell$ ⊓ 𝑚.
- The *Set Cases* rule: if $\Gamma$ ∪ {S = ∅} $\vdash$ $\phi$ : $\ell$ and $\Gamma$ ∪ {$\neg$(S = ∅)} $\vdash$ $\phi$ : 𝑚 then $\Gamma$ $\vdash$ $\phi$ : $\mathcal{C}$.

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
