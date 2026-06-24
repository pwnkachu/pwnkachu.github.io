---
title: Tamarin SAPIC+
description: This pages serves as a note for SAPIC+ (Stateful Applied Pi-Calculus) for procotol specification modeling.
---

# SAPIC+
Stateful Applied PI-Calculus is a security protocols verification platform, that is used to enable verification on multiple tools (eg: Tamarin or ProVerif). It is based upon Applied Pi-Calculus that is a formal language used for modelling and verify security properties of IT protocols. Applied Pi-Calculus is an extension of Pi-Calculus, that was initially used to describe the exchange of messages. Applied Pi-calculus introduced Processes, Communication channels and cryptographic functions. SAPIC+ also introduces a global state to Applied Pi-Calculus, enabling formal verification across multiple tools.

# How to
We can use SAPIC+ to model protocols as processes. In Tamarin we can use both multiset rewrite rules and SAPIC, althogh it's not raccomended due to the fact that the SAPIC+ process is transalted into a set of rules and their interaction depends on how the translation process is defined.

SAPIC+ defines the following features ([Tamarin documentation](https://tamarin-prover.com/manual/master/book/006_protocol-specification-processes.html)):

```
<P,Q> :: =  (elementary processes)
    new n; P                .. binding of a fresh name n to process P
  | out(t1,t2); P           .. output of t2 on a channel t1
  | out(t); P               .. output of t on the public channel
  | in(t,x);~P              .. input on channel t binding input term to $x$}
  | in(x);~P                .. input on the public channel binding to $x$}
  | if cond then P else Q   .. conditional
  | let t1 = t2 in P else Q .. let binding
  | P | Q                   .. parallel composition
  | 0                       .. null process
```

From the tamarin documentation we can see a basic example of a process that picks an encryption key, waits for an input and performs decryption:

`new k; in(m); out(senc(m,k))`

Processes can also perform branch execution with:
```
if cond 
    then P 
else Q 
```
This will execute either P or Q, depending on whether cond holds.
 Most frequently, this is the equality check of form t1 = t2, but one can also define a predicate using Tamarin’s security property syntax (lemma).

## Let-bind

One can use `let` for improve reading. For example one may bind a cryptographic function to a variable in order to improve readability.

The other option is to bind destructors (operations that will deconstruct a message into an other, such as `adec` that is the assymetric decryption) and perform logical checks on the failure of that operation. Tamarin documentation provides an example:
`adec(aenc(x.1, pk(x.2)), x.2) = x.1`

The operation result in a success only if the private key is the one associated with the public key.

```
let m = adec(ciphertext, my_key) in
    // True logic
else
    // False logic
```

The difference from Multiset Rewrite Rules is that if we specify a deconstructor in a rule and the operation results in a failure, the rule will be not executed. In the case in which we would like to model edge-cases such as authentication failure in a Protocol we are suggested to use SAPIC+, that will automatically be traduced into multiple rules that respects the if else logic. Let-bindings permits *pattern matching*, this is very useful to model our protocol message exchange:
```
in(x); 
let <'pair',y,z> = x in out(z);
out(z).
```

In the previous example we simply omit the else execution, if pattern matching results in a failure the execution simply stops.

### Explicit Pattern Matching
Pattern matching is explicit, one can bind values to a variable or perform a check on an existing variable and bind it to a new one:

```
let <y, =z> = x
```
This will bind a value to y and perform a checks, the value binded to z is checked against an existing variable x.

## Parallelism

We can model the parallel execution of two processes as `P | Q`. This can be used to model the execution of two participants in a protocol. The original syntax is extended with:

```
<P> :==    (extended processes)
  | event F; P  .. event   
  | !P .. replication
```

The event construct can be seen as an action (it will be traduced into an action) and are useful to state and check security properties. !P is the infinite replication of P which allows an unbounded number of sessions in protocol execution (infinite amount of processes P | ... | P ...)


## Global State

With SAPIC+ we can also store, retrieve and modify a global state:

```
<P,Q> :==        (stateful processes)
  | insert t1, t2; P .. set state t1 to t2
  | delete t; P  .. delete state t
  | lookup t as x in P else Q ; P  .. read state t into variable x
  | lock t; P .. set lock on t
  | unlock t; P .. remove lock on t
```

One can model the environment of a single entity as:
`insert <'EntityType',entity_id, 'store'>, data; P`


### Inline Multiset Rewrite Rules
`[l] --[a]-> r; P` is defined as a valid process. Those embedded rules applies if the `[l]` facts holds and the process is reduced up to this rule.


## Modeling execution

One can model local progress of a process. It is importatn to know that there exist a 0-process that is a process that does nothing. A process can be reduced unless it's waiting for an input, it's under replication or is already a 0-process. We can model *time-outs* with:

`option: translation-progress`

In which a process must reduce to either P or a timeout one:

`( in(x); P) + (out('help');0)`

If the input message is received the it will produce regulary (P). The right side is used to model a timout recovery protocol. To model this SAPIC+ uses hidden events. When a process reaches this states (transitions to two possible processes) SAPIC+ emits a `ProgressFrom` event, that needs to be associated to a `ProgressTo`, this will always result into an evolution of the process from the engine persepctive. 

## Isolated Execution Environments

An Enclave (Trustes Execution Environment) is the execution of a process within a secure, isolated environment. This ensures that sensitive operations are protected even if the sourrounding systems is compromised (eg: Intel Software Guard Extension SGX). To model those technologies SAPIC+ uses:

`let A = (...)@loc` in which the @loc identifier is used to tell that the process A is an enclave. Those environment must be able to prove that they performed an operation (due to the facts that external process shouldn't be able to read it's internal state), this case is modele through:

```
let x = report(m) in ...
```

and 
```
if input = check_rep(rep, loc) then ...
```

#### Attacker in enclave perspective
An attacker can use an enclave, this he can produce reports for its' onw processes or locations. The only thing that an attacker shouldn't be able to do is to generate a report for any @loc. The user can defined a set of unstrusted @loc:

```
predicates:
Report(x,y) <=> phi(x,y)
```

We use a predicate to model a logic function that represents the report function. X is the data to be reported and y the location. In order to know if the Report is true, the phi() function must be true. An example of usage is:

```
predicates:
Report(m, loc) <=> Compromised(loc) | IsAttackerNode(loc)
```
The attacker can generate a fake Report() if and only if that location is compromised or is an attacker location.


### Process declaration using let

One should model the processes around the role that they represent in the protocol. This can be done with let:

```
let Webserver (identity) = in(<'Get',identity..>); ..

let Webbrowser () = ..

(! new identity !Webserver(identity)) | ! Webbroser
```

and let also permits nested processes:

`let B() = A() | A()`

#### Types declaration

One can define types to avoid modelling errors. This will not affect the attacker and those type will be disregarded after translation into MRR.

```
Types can be declared for function symbols:

functions: f(bitstring):bitstring, g(lol):lol,
            h/1 // will implicitely typed later.

for processes:

new x:lol;                             // x is of type lol now
new y;                                 // y's type will be inferred
out(f(y));                             // now y must be type bitstring ...
// out(f(x));                          // fails: f expects bitstring
out(<x,y>); out(x + y); out(f(<x,y>)); // lists and AC operators are type-transparent
out(h(h(x)));                          // implictely types h as lol->lol
// out(f(h(x)));                       // fails: as h goes to lol and f wants bitstring

and subprocesses:

let P(a:lol) =
```

### Exporting

Tamarin documentation also describes how quesies and lemmas can be exported. It's important to note that one can decide the output mode for external tools such as proverif and deepsec. One can also define the output tool in a lemma:

`lemma secrecy[reuse, output=[proverif,msr]]:`