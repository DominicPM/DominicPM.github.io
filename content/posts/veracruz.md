---
title: "Privacy-preserving collaborative compute with Veracruz"
date: 2021-04-237T23:11:54+01:00
draft: false
tags: ["veracruz", "confidential-compute"]
---

[Veracruz](https://github.com/veracruz-project/veracruz) is a project that I've been working on for a while internally within Arm Research.
Recently, Veracruz was released as an open-source project on Github, with all development and design discussions moving into the open.
This was a precursor to the project being adopted by the Linux Foundation's [Confidential Computing Consortium (CCC)](http://confidentialcomputing.io), which I'm pleased to say finally happened a few weeks ago, at the beginning of April.

In this post, I'll explain a little bit more about the project and what its motivation.

## Protecting data

Data exists in one of three modes: *at rest*, *in transit*, and *in use*.
Generally speaking, we know how to protect data when it is at rest, using full-disk encryption technology built around standard block ciphers like the Advanced Encryption Standard (AES).
Moreover, we understand how to protect data when in transit, that is when it is being sent from computer to computer, using standard transport layer security protocols like TLS.

Where things get interesting is when trying to protect data when in use, that is, when it is being actively computed on.
Cryptographers have made some great strides over the last 15 years in defining new encryption schemes that protect data when in use.
These include so-called *fully-homomorphic encryption schemes*, the first of which was discovered by Craig Gentry, which allow us to compute a function directly on ciphertext without first requiring decryption.
Moreover, in collaborative settings, cryptographers have defined a number of protocols for *secure multi-party computations*, wherein a group of mutually-distrusting principals wish to compute a function over their combined private inputs without revealing their inputs to any other principal.
Using these tools, we are able to compute functions on data without requiring that the data be decrypted (in the case of homomorphic encryption) or without necessarily sharing raw data (in the case of secure multi-party compuations).

All is well.
Data leaks are a thing of the past.

Or maybe not.

These *Advanced Cryptography* tools tend to have a common set of frailties that make them hard to deploy in real world settings.
Perhaps &mdash; most infamously &mdash; they are *slow*: computing directly on ciphertexts with homomorphic encryption schemes incurs significant performance penalties when compared to native execution speeds.
More than that, they also tend to be quite hard to deploy in practice, tend to be brittle in the face of shifting business requirements, and perhaps most importantly of all are almost impossible for non-cryptographers to actually understand or implement securely.
This latter point is probably more important than it first appears, as &mdash; by and large &mdash; there are relatively few production grade implementations of these cryptographic tools; even if you were lucky enough to find an algorithm that perfectly matches your business requirements, chances are it would be up to you to implement it from scratch using research papers as a guide.

At this point, I should not that cryptographers have made really quite impressive strides in boosting the performance of many of the cryptographic techniques mentioned above.
The performance penalty associated with homomorphic encryption has decreased by many orders of magnitude over the last decade, for example, to the point that it is now feasible to use it to protect small problems of interest.
But, there's still a way to go.
Is there an interim solution that we can use to protect data when in use?

## Hardware support for Confidential Computing

A recent trend in the hardware industry is the addition of explicit support for *Confidential Computing*.
This support comes in the form of a protected execution environment &mdash; variously called a *Secure Enclave*, a *Realm*, a *Trusted Application*, a *Protected Virtual Machine*, or similar &mdash; that provides strong protections to code and data hosted within.
Concrete examples of this trend include Intel's Software Guard Extensions (SGX), AMD's Secure Encrypted Virtualization (SEV), and the recently announced Arm Confidential Compute Architecture (CCA).

(Given the large number of names associated with these technolgies &mdash; and the fact that some names are now strongly associated with a particular technology &mdash; I will use *strong isolation technology* to refer to a generic enclave-like technology, and use *isolate* to refer to a generic protected execution environment, henceforth.)

Up close, each strong isolation technology adds its own specific twist on the theme.
Howver, from a distance, it's still possible to speak in gross generalities about each of them and the features that they tend to provide:

- *Confidentiality and integrity guarantees*: code and data hosted within an isolate are provided strong guarantees against monitoring or interference by an external adversary. 
- *Protection in the face of a strong attacker*: system software, such as the operating system and hypervisor, are explicitly outside of the trusted computing base.
In short, an isolate is &mdash; as the name suggestes &mdash; completely isolated from the operating system and any other software running on the host machine, including other isolates.
- *Associated remote attestation mechanisms*: remote attestation is a cryptographic technique which allows a skeptic to challenge the authenticity of an isolate and the integrity of any software loaded within it.
Using remote attestation, we can check, and receive with a high level of assurance, that software claiming to be running inside an isolate on a remote machine is actually running inside an isolate (and not emulated), and was configured and loaded with the software that we expect.

Putting the three properties above together, we can use strong isolation technologies to create a safe, neutral ground within which a collaborative execution can take place.
Any such execution is protected by the isolate's confidentiality and integrity guarantees, even against an adversary that is able to subvert the exceptional powers of the host operating system.
Moreover, using remote attestation, we are able to check that the isolate that we are interacting with, and which is claiming to protect our computations, is indeed authentic and was also loaded with the software that we expect.

Essentially, strong isolation technologies provide what's needed to protect data when in use, without necessarily having to use advanced cryptographic techniques.
Note that using hardware, instead of cryprographic, to protect data in use is not entirely without its downsides: we'll necessarily have to trust the correctness and security of more than we would with a cryptographic solution.
I'll come back to this point, later.

## Veracruz

*Veracruz* is a framework &mdash; a collection of protocols, tools, and a trusted runtime &mdash; for defining and deploying privacy-preserving collaborative computations.
In the most general case, a computation defined with Veracruz consists of *N* distinct *data providers*, each of which has an associated data set that they wish to contribute as an input to the computation.
Working with the data providers is a distinguished *program provider* who is charged with providing the computation that will transform the data provider's inputs into an output.
Together, the data and program providers will outsource their computation to another party, which we call the *host*, and who will contribute the computational power to affect the collaborative computation.

Together, the data providers, program provider, and host are all *mutually distrusting*.
Each data provider does not wish to reveal their data sets to each other, nor to anybody else for that matter, and the program provider does not necessarily wish to reveal their program binary, neither, as it may constitute sensitive IP.
Moreover, the host is explicitly untrusted by both program and data providers, and generally also wants to place strict limits on the capabilities of the program over the course of its execution lest a malicious program provider attempt to run malware on the host's machine.

Concretely, Veracruz provides a *trusted runtime*, the source of which is [open](https://github.com/veracruz-project/veracruz) and auditable by anybody who cares to look, which will execute inside of an isolate spawned on the host's machine.
The trusted runtime, as the name suggests, must be explicitly trusted by everybody involved in the computation, hence the open-source nature of the runtime's codebase is not merely a development decision on our part, but a vital part of the Veracruz trust story.
In particular, everybody needs to ensure that the trusted runtime has not been "backdoored" or tampered with by any malefactor before engaging further in the computation.

Each Veracruz computation is parameterized by a *global policy*.
The policy is a configuration file that specifies various bits of important information about the computation itself, namely *who* is involved (where principals are identified by cryptographic certificates) and *what* they will be doing as part of the computation.
Specifically, each principal in the computation is assigned a set of *roles* by the global policy, detailing whether they are to provide data, provide the program, or receive a result from the computation.
Note that one principal may actually take on many roles.
For example, depending on the particular computation, a data provider may also be tasked with providing the program, and may also be able to receive the result of the computation.
As a result of this, everybody must also have audited the policy, and understand its consequences, before enrolling any further in the computation.

Together, the policy and the program provider's program define the "shape" and "means" of the collaborative computation.
Varying either one changes the nature of the computation.
Note that steps taken to agree on the text of policy are out-of-scope for Veracruz, we simply assume that there is some prior "out of band" communication that allows all participants to agree on one prior to the computation taking place.

Assuming that the policy is acceptable to all involved, the first step in computing a result with Veracruz is for the program provider to challenge the authenticity of the isolate spawned by the host, and check that the content of that isolate is indeed the runtime that the program provider audited, and a malicious host had not "backdoored" it after audit.
To do this, the program provider uses a *remote attestation* procedure to obtain an *attestation token* that contains a *measurement* &mdash; or cryptographic hash &mdash; of the software loaded into the isolate.
This attestation token can be passed to an external attestation service, such as the Intel Attestation Service (IAS) associated with Intel SGX, which will either authenticate the token as originating from a legitimate isolate, or reject it.
Moreover, assuming that we can produce *reproducible builds* of the trusted runtime, the measurement contained within the attestation token can be compared against an independent measurement of the runtime that the program provider is themselves able to independently produce.
Once authenticated, providing the program provider trusts the attestation service, the token serves as strong evidence that the host has indeed started a legitimate isolate (of a particular kind) and that the isolate contains a legitimate Veracruz trusted runtime.

Once this dance is completed, the program provider provisions their program directly into the trusted runtime loaded within the isolate.
To do this, they establish a TLS connection that terminates *inside* the isolate.
As a result, from the perspective of the untrusted host, the traffic between the program provider and the trusted runtime is completely opaque.

Once this step is completed, each of the data providers follows the same dance that the program provider performed, first challenging the authenticity of the isolate using remote attestation, and then comparing the measurement of the trusted runtime against a precomputed hash, if desired.
Once convinced, each of the data providers then provisions their respective secrets directly into the trusted runtime, again using a TLS link which obscures the secret from the untrusted host. 
However, data providers may engage in an additional step before provisioning their secrets, namely they may request from the runtime the hash of the binary that the program provider provisioned into the runtime previously.
In cases where the program provider chooses to intentionally declassify their program, revealing it to the other participants before the computation begins (we'll discuss some cases where this makes sense, below), this allows the data providers to ensure that the program provider provisioned the program that they promised they would.
Note that, along these lines, both data providers and program provider can also request details of the global policy being enforced by the trusted runtime, again to make sure that the policy they agreed to is in fact the one that is being enforced.


By now, both the program and all of the input data sets are now provisioned inside of the trusted runtime.
Note that one interesting aspect of Veracruz is that we support a range of different isolation mechanisms.
At the time of writing, the trusted runtime could be loaded into an Arm TrustZone trusted application, an Intel SGX Secure Enclave, an AWS Nitro Enclave, or make use of software isolation mechanisms afforded by the high-assurance seL4 hypervisor.
As a result of this, we need some way of abstracting over the particular instruction set in use by the isolate spawned by the host.
For this reason, in Veracruz, the program provider's program is supplied as a WebAssembly binary, which serves as an architecture-independent executable format.
Moreover, by using WebAssembly, we can control the capabilities of the program provider's program, providing it with only the capabilities needed to express a range of computations of interest.
To a first approximation, you can assume that the program is only capable of expressing a *pure* function of the inputs from the various data providers, modulo the ability to generate some random data.
Specifically, the program is prevented from opening any file, or otherwise producing any observable side-effect, on the host's machine.
This both protects the host, and also provides a defence against an untrusted host and program provider from colluding to steal secret inputs. 

Once all secrets are provisioned into the isolate, the computation can go ahead and compute a value as output (or diverge).
After this is done, the principals able to receive the result, per the global policy, by establishing a TLS link that again terminates inside the isolate.
Again, this ensures that the untrusted host is unable to view or manipulate the result of the computation as it is transferred out of the isolate to each of the result receivers.

## Use-cases

The description of Veracruz in the last section is fairly abstract, but can be better understood by explaining how some concrete problems can be solved using the framework.

### Privacy-preserving machine learning

Alice and Bob are two rival online retailers who each have a private dataset capturing click-through data for customers on their respective webstores.
The two wish to produce a machine learning recommendation engine to show relevant products to their customers and drive new sales.
However, both realize that the size of their respective datasets makes it unlikely that they can produce decent recommendations, but by combining the two, they can produce a much better recommendation engine than either could separately.
However, given that Alice and Bob are both rivals, neither wishes to reveal their private datasets to the other.
To produce this machine learning model over the two datasets, Alice and Bob will use Veracruz.

First, the two must agree on a global policy.
To do this, arbitrarily one of them will be tasked with producing (or supplying) the program which wil implement the machine learning algorithm of interest, assuming that the inputs are packed into some mutually-agreeable data format, and so on.
Let's assume Alice is tasked with the producing of this program.
Moreover, both will be listed as providing data inputs, and also received the output from the computation, by the policy.
After this, Alice goes ahead and implements the machine learning model, and then declassifies the program by sharing it with Bob, so he can audit it and check that it is indeed what was agreed.

Once Bob is happy with the program that Alice has produced, Alice can go ahead and provision the program into the Veracruz trusted runtime, hosted within some isolate spawned on a mutually-agreeable host's machine, after following the remote attestation song-and-dance detailed above.
Once done, both Alice and Bob then also provision their private datasets &mdash; the inputs to the computation &mdash; again after performing this same remote attestation procedure.
Importantly, at this stage, Bob should also check that Alice had indeed provisioned the correct machine learning algorithm that he had previously audited (and if not, bail out of the computation).

At this point, everything is in place to go ahead and apply the machine learning algorithm to the inputs.
As both Alice and Bob are specified as being able to receive the result by the global policy, once the computation completes they can both connect to the isolate using a TLS link to retrieve it.
Note that neither Alice nor Bob ever see the secret input of other, nor does the host of the computation, as all secrets are only ever in one place inside the isolate which is assumed to have strong confidentiality guarantees that protect code and data hosted within.

### Other potential use-cases

It turns out, with a little thought, that a lot of collaborative computations fall into the general pattern of *N* data providers trying to collaborate with a program provider to achieve some desired effect.
For example, with a little thought, it's possible to apply Veracruz to problems like:

- Private set-intersection, or private set intersection-sum problems,
- Privacy-preserving genomics,
- Privacy-preserving social graph analytics,
- Protecting commercially-sensitive algorithms and other IP,
- Secret auctions, polls, surveys, and similar,
- Securely offloading computations from computationally-weak devices,
- Oblivious routing in maps,

...and many more.

## Comparison with cryptographic techniques

So, how does Veracruz compare to pure cryptography?

One disadvantage of Veracruz is the size of the trusted computing base when compared with pure cryptography.
Namely, the trusted Veracruz runtime (including the WebAssembly execution engine) must be trusted, as must the remote attestation protocol and external attestation service.
Moreover, with Veracruz, we provide no explicit protection against side-channel attacks at present.
There are several opportunities for a malicious participant (especially the program provider) to insert side-channels into the program, for example, in order to surreptitiously exfiltrate secrets from the isolate.
If participants are concerned about the risk of side-channels, then they must insist on the program provider declassifying their program as a precondition of taking part.

On the other hand, Veracruz is flexible in a way that cryptography typically is not.
Focussing on one of the suggested use-cases above, we could use Veracruz to obliviously compute directions from point-to-point in a map, working directly with OpenStreetMap data, for example, inside the isolate, making use of existing libraries for graph shortest path algorithms to complete the task.
What's more, Veracruz computations can be written in a range of high-level programming languages, owing to our use of WebAssembly, making use of standard programming tools and idioms.
This is in contrast to cryptographic approaches, which either must be compiled to some special-purpose representation (e.g. arithmetic circuits) which make computations hard to develop and debug.
Finally, computations can be freely composed in Veracruz just by combining programs and subprocedures.
In contrast, safely composing protocols for secure multi-party computations is a challenge in itself.

## Reading more

To find out more:

- For more introductory material on Veracruz you can see my recent talk in the Hardware-aided Trusted Execution Environment DevRoom at FOSDEM 2021.
A recording is available [here](https://video.fosdem.org/2021/D.hardware.trusted/tee_veracruz.mp4).
- For more in-depth material about how Veracruz handles remote attestation you can see Derek Miller's recent talk at Linux Conf Australasia 2021 and the Open Confidential Computing Conference (OC3) 2021.
Recordings are available [here](https://www.youtube.com/watch?v=CoI4NLmQTqE) and [here](https://www.youtube.com/watch?v=6njKU9v1kEg&list=PLEhAl3D5WVvSKTp9jD6lV67zuFR8lUZVa&index=12).
- An upcoming (at the time of writing) Confidential Compute Consortium (CCC) webinar from members of the Veracruz team (including me).
Click [here](https://confidentialcomputing.io/webinar-veracruz/) for details.

Finally, check out the [Veracruz homepage](https://veracruz-project.github.io), which posts ongoing developments in the project, and the [main Github repository](https://github.com/veracruz-project/veracruz) for the project.
We are always looking for collaborators to work with us, and the Github issue tracker for the project contains many ideas for new contributors, including many that are suitable for newcomers to the project.
Details of public weekly technical discussions, and our Slack channel hosted by the Confidential Compute Consortium, are available in the README in the repository. 
