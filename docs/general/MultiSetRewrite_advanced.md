---
title: Tamarin Protocol Multiset Rewrite Advanced features Guide
description: Personal reference guide of the official Tamarin protocol modeling chapter with advanced features focus.
---


# Embedded restrictions

Restriction are used to reduce the set of traces to consider in the protocol analysis. They can be used to model branch behaviour of protocols. A very simple example is the Equality restriction used to check if the verification of a signature is true or not:

```
restriction Equality:
  "All x y #i. Eq(x,y) @i ==> x = y"

[...]
--[Eq(verify(sig,m,pkA),true) ]->
[...]
```

Tamarin doc provides a list of common restrictions: [Restrictions](https://tamarin-prover.com/manual/master/book/007_property-specification.html#sec:restrictions)
such as Uniqueness, Equality, Inequality, OnlyOnce ....

One can also use embedded restriction, that are applied directly into the Action Facts of a rule:

```
rule X:
    [] --[ _restrict(formula) ]-> [ ]
```

is then translated to:

```
rule X:
    [ left-facts] --[ NewActionFact(fv) ]-> [right-facts]

restriction Xrestriction:
   "All fv #NOW. NewActionFact(fv)@NOW ==> formula

```

# Diff() (Observational Equivalence)

Observational Equivalence properties reason about two systems (two istances of a protocol) by showing that an attacker cannot distinguish the instances. This can be used to express additional properties such as privacy and  indistinguishability.
Tamarin documentation provides a voter example in which the attacker shouldn't be abel to know if an instance of the voter A voted for x or y in two different instances.

```
rule Example:
    [ Fr(~ltk)
    , Fr(~a)
    , Fr(~b) ]
  --[ Secret( ~b ) ]->
    [ Out( pk(~ltk) )
    , Out( ~a )
    , Out( aenc( diff(~a,~b), pk(~ltk) ) )
    ]

lemma B_is_secret:
  " /* The intruder cannot know ~b: */
    All B #i. (
      /* ~b is claimed secret implies */
      Secret(B) @ #i ==>
      /* the adversary does not know '~b' */
      not( Ex #j. K(B) @ #j )
    )
```
In this protocol Tamarin will otuput that the attacker can know if an instance has voted x or y.

# Lemma annotations

A lemma can be associated with various notations that are used to change its menaing.
The following notations are presented:

- `source`: The lemma's verification will use induction and the lemma will be verified using the Raw Sources and will be used to generete Refined Sources, that are used for verification of all the other lemmas. Those lemma are used to help Tamarin in the refining process of Raw Sources (Tamarin doesn't know where a message was generated) in order to avoid infinite loops and state explosion.

- `use_induction`

- `reuse`: A reuse lemma will be used in the proofs of all lemmas syntactically following it (excpet sources). Reuse lemmas are ignored in the proof of the equivalence lemma (modeled using diff()).

- `diff_reuse`: A lemma marked diff_reuse will be used in the proof of the observational equivalence lemma. Diff_reuse lemmas are not reused for trace lemmas.

- `left and right`: In equivalence mode we have two protocols, the left and the right instance. If we want to consider a lemma only on one of those instantiation we can annote it with left or right.

-`hide_lemma`: Used to hide a lemma to pther specific lemmas.

# Heuristics

Tamarin uses a constraint system in order to chose how to (try to) verify a lemma. In order to Tamarin usually uses `s`(smart) heuristic that is used to solve most of modern protocol's properties. But for complex protocols we may want to define a custom heuristic to prevent infinite loops (in addition to source lemmas). We can specify an heuristic from the CLI with the `--heuristic=` command or a custom heuristic for every lemma with `lemma x [heuristic=C]`. We can also use a global heuristic defined at the start of the file and it's applied to every lemma of the file unless there is an heuristic specified by the CLI (most prioritized) or in a lemma.

The heuristics are:

- `s`: Default heuristic that works very well in a vast amount of situations. Prioritizes chain constraints, disjunctions, facts, actions and adversary knwoledge in that order. Contraint marked Probably constructed and Currently Deducible by the GUI are lower priority.

- `S`: Like the smart ranking but does not delay the solving of premises marked as loop-breakers. Those are determined "using a simple under-approximation to the vertex feedback set of the conclusion-may-unify-to-premise graph."

- `c`: Conservative ranking. It solves constraints in the order they occur. Often leads to large proofs because some constraints are not worth solving.

- `C`: Like C but without delaying loop breakers

- `i`: The priority of proof methods is similar to the ‘S’ ranking, but instead of a strict priority hierarchy, the fact, action, and knowledge constraints are considered equal priority and solved by their age. This is useful for stateful protocols with an unbounded number of runs, in which for example solving a fact constraint may create a new fact constraint for the previous protocol run.

- `I`: Like i but without delaying loop breakers

## Tactic
We can use a tactic to specify a custom heuristic for the presort phase and priority inside the source code. A tactic is used to defined the exact order for solving the constraint system.

```
tactic: Tacticname
presort: s
prio:
  regex "!KU\( ~.* \)"
  regex "State_.*"
deprio:
  regex "In_S\( .*"
```

The presort field is optional and will be set by default to s. 
`prio` is a waterfall structure, so the first row has the higher priority. 
`deprio` works as the prio one but with the opposite goal since it allows the user to put the recognized proof methods at the bottom of the ranking

To tell Tamarin what are action it need to prioritize we can used predefined functions:

- `regex` will search a regex inside the objectives of the constraint system. Parenthesis needs to be escaped.

- `isFactName` Will lookup the name of the Tamarin objective and check it with its parameter.

- `isInFactTerms`  will look in the list contained in the field FactTest whether an element corresponding the parameter can be found.

It's important to know that we can define multiple conditions for a prio block. We can do that with `|, &, not`operators:

```
prio:
    isFactName "ReceiverKeySimple"
prio:
    regex "senc\(xsimple" | regex "senc\(~xsimple"
prio: {smallest}
    regex "KU\( ~key"
}
```

another example is the one in which we prioritize input from the network and avoid lecture from the internal state facts:

`regex "In" & not (regex "In_S")`

### {smallest}

If the same prio find multiple options that satisfies the same prio (or regex) we can define the order to solve those with {smallest}. Without smallest we mantain the basic presort order. With {smallest} option we decide to count the length of the option and order them from the shorter to the longest one.

```
KU( ~k ) 

KU( senc(~msg, ~k) )

KU( aenc(senc(~msg, ~k), pk(~other)) )
```
The length of the option is seen as a metric for its complexity.


# Oracles


# Additional Channels mode

We can limit the attacker control over the protocol agent's communication channels by specifying channel rules, that will model channels instrinsic security properties. Those are additional rules to add to our source code that model different type of channels:

- `Conf()` A confidential channel is one in which we can send a message and we are assured that only the intended receiver (must be specified) will be able to read the message. The attacker can still inject and craft messages to the channel in order to impersonate another entity.

```
rule ChanOut_C:
        [ Out_C($A,$B,x) ]
      --[ ChanOut_C($A,$B,x) ]->
        [ !Conf($B,x) ]

rule ChanIn_C:
        [ !Conf($B,x), In($A) ]
      --[ ChanIn_C($A,$B,x) ]->
        [ In_C($A,$B,x) ]

rule ChanIn_CAdv:
    [ In(<$A,$B,x>) ]
        -->
        [ In_C($A,$B,x) ]
```

- `Auth()` The adversary can learn everything sent on an authentic channel. The adversary cannot change the sender of the message neither the message itself.

```
rule ChanOut_A:
    [ Out_A($A,$B,x) ]
    --[ ChanOut_A($A,$B,x) ]->
    [ !Auth($A,x), Out(<$A,$B,x>) ]

rule ChanIn_A:
    [ !Auth($A,x), In($B) ]
    --[ ChanIn_A($A,$B,x) ]->
    [ In_A($A,$B,x) ]
```

- `Secu()` A channel that is both confidential and authentic. An attacker can still store a message sent over a secure channel for replay at a later point in time.


```
rule ChanOut_S:
        [ Out_S($A,$B,x) ]
      --[ ChanOut_S($A,$B,x) ]->
        [ !Sec($A,$B,x) ]

rule ChanIn_S:
        [ !Sec($A,$B,x) ]
      --[ ChanIn_S($A,$B,x) ]->
        [ In_S($A,$B,x) ]
```

If we want to avoid replay attacks we can use a `Sec($A,$B,x)` linear fact instead of a persistant one