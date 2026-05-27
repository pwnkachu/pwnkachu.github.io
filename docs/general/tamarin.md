---
title: Tamarin guide
description: Personal reference guide Tamarin prover symbolic analyzer.
---

# Tamarin Overwiev

## What is Tamarin?

Tamarin is an open source analysis tool for security protocols. It can verify the security of a protocol given the specification of the protocol, the threat model of the adversaries and the security properties that the protocol should assert. 

## What can Tamarin do?

Tamarin can either verify the protocol or provide some counterexample of attacks.
Tamarin che assure that a security property holds for every possible execution of the protocol.

## Formal Methods

In order to provide Tamarin the specification of the protocol, the model of the attacker and the security property to assert we need a formal method. Formal methods are a part of computer science that provide a semantyc to rigorously describe a system, in order that one can reason about the system using rigorous logic and mathematic. Verification tools (as Tamarin) are either classified as Theorem Prover or Model checkers:

- Model checkers are automated formal verification tool that will check all possible states of a system to detect if a property is not hold (a state that should not be reachable) and will provide a counterexample (the stack trace) if the property is not holded. 

- Theorem provers are tool that uses explicit proofs using formal logic, such as the first order logic. 

Tamarin is an automated and interactive theorem prover that uses the simplified modelling logic of model checkers. When Tamarin is executed we can expect the following results:

- Tamarin finds a behavior satisfying all the constraints. This represents an attack
on the protocol.

- Tamarin produces a proof that the constraints are inconsistent. This means that
there is no possible attack and the protocol is therefore secure. This proof can be
produced automatically, or interactively with user support.

-  Tamarin fails to terminate and so the result is inconclusive. We do not learn
which case holds: whether the protocol is secure or there is an attack.


## Modeling languages

Tamarin supports two modeling languages that are used to model the protocol, the adversary, and the required properties. 

#### Multiset Rewriting
This language is Turing-complete and can formalize any system and adversary. These are modeled using a transition system with a computable transition relation. 

To provide a better understanding of multiset rewriting, here is an explanation of its core components.

Tamarin uses a set of **facts** to model the system's state, which can represent statements such as *"Agent A has sent message X"*.

A **transition system** is a mathematical model composed of:

* **States:** All the possible configurations (multisets) of these facts.
* **Transitions:** The steps from one state to another, governed by the multiset rewrite rules.

Every rule consumes the facts present on the left-hand side ($L$), generates new ones on the right-hand side ($R$), and optionally registers a trace action ($A$):

$$L \xrightarrow{[A]} R$$

The **transition relation** is the set of all pairs of states $(\text{State}_A, \text{State}_B)$ where a legal transition exists dictated by the rules. Because this relation is computable, Tamarin can deterministically compute all possible next states from any given state by applying pattern matching (unification) to the rules.

Basically, Tamarin operates based on three core concepts:

* **The Current State:** Represented as a multiset of facts. Tamarin tracks all the facts that exist at a given moment (e.g., State_1 might contain: *"Agent A has started a session"* and *"Key K is secret"*).
* **The Available Rules:** These are the user-defined multiset rewrite rules written in the source code. They define the legal actions and state transitions allowed in the protocol.
* **Pattern Matching (Unification):** Tamarin checks which rules have a left-hand side ($L$) that matches the facts available in the current state. If a match is found, the rule is applied: the matching facts are consumed (removed from the state), and the new facts specified on the right-hand side ($R$) are generated, effectively creating the next state.

### First-Order Logic-Based Language

Tamarin utilizes a second modeling language to specify the system's security properties, which are written as **Lemmas**. This language is based on sorted first-order logic and supports quantification over **time points**.

??? "First-Order Logic"
    Also known as predicate logic, first-order logic is a formal language used to represent systems and reason about them mathematically. It extends propositional logic by using quantified variables (such as "for all" $\forall$ or "there exists" $\exists$) and predicates. For example, the statement *"All humans are mortal"* is formalized as:
    $\forall x . (\text{Human}(x) \rightarrow \text{Mortal}(x))$
    Where $x$ is a variable, $\forall$ is the universal quantifier, and $\text{Human}$ and $\text{Mortal}$ are predicates.

Every time a multiset rewrite rule is executed, it can log a specific trace action (an event tag). Tamarin uses the logic language to assign a **temporal variable** (e.g., $\#i$, $\#j$) to these actions. 

This logic is interpreted over the **finite traces** (the execution history) of the transition system. By analyzing these timelines, Tamarin can evaluate complex temporal formulas. For instance, it can mathematically verify properties such as: 

> *"If an initiator $A$ accepts a session key at time point $\#i$, then the adversary must not have learned that key at any other time point $\#j$."*

Because this language is highly expressive, it allows users to model nuanced security properties (e.g., forward secrecy) and test them against realistic, highly capable adversaries, such as malicious insiders or attackers who can compromise specific keys at precise moments in time.


