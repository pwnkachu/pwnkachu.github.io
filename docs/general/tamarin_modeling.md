---
title: Tamarin Protocol Modeling Guide
description: Personal reference guide of the official Tamarin protocol modeling chapter.
---

## Modeling multiple instances of a protocol

The parties that run a protocols will often run multiple istances on the same machine concurrently, those istances can be identified as a thread.

```
 State(~threadID, $A, 'Waiting'), In(msg) ]
--[ ReceivedMessage($A, msg) ]->
[ State(~threadID, $A, 'Done') ]
```

In this scenario we have modeled a rule where there is a protocol thread that waits for a msg on the network and do a transition to the "Done" state after receiving the message.

To specify the transiction system, Tamarin uses three syntactic categories:

- **Terms**: That represent protocol messages, bitstrings or variables.

- **Facts** That are used for 2 purposes:

    - Used to record information in the transition system's current state and to perform checks on it. `State(), In(), Out()` are examples of Facts. 

    - Used to log performed actions, such as `ReceivedMessage()`, those are relevant for perperty specification.

- **Rules**: That are used to model the transition system's possible transitions.


## Symbolic Modeling

In order to rapresent the messages that are manipulated and shared by protocols, Tamarin uses *symbolic modelling* that are a term-base representation of the messages (that in the real world are bitstring) where the term rapresent how the messages are constructed by applying a cryptographic function to data. 

An example is the application of an Hash H to a string (that could be a message in the protocol). In Tamarin this application is not the concrete data that would result by applying the hash function to it, but it's a term like: `H("Hello World")`. The symmetric encryption of a message with some key k is defined similarly: `senc(m, k)`.

With this method Tamarin can reason about *classes of messages* rather than focusing on each concrete instantiation. The alternative to symbolic modeling is cryptographic modeling.

# Terms

Tamarin uses *sorting discipline* (The system types used by Tamarin) to categorize all the terms (that we recall are the elements passed around as data or keys). This is done to ensure that the terms are used correctly and there isn't any type flaw. There is a top sort `msg` and two incomparable subsorts `fr` and `pub` of the top sort. For those sorts there are two associated sets of infinite values that are:

- Fresh values: Used to model freshly generated random values, key, nonces (unguessable by the adversary).

- Public costants: Which are used to model guessable informations such an agent id. Public costants are written as anyone would expect: 'Agent 1' is a public costant.

There is also an infinite set of *Variables* for each sort. Variables have names athat are unquoted text string, such as : `myvar` and can be followed by a type declaration. *Variables do not have global scope* , so we can only have local variables.

- Public variables: are defined as a variable with a `$` prefix. `$C and $C` are the arbitrary client or server names.

- Fresh variables: are defined as a variable with a `~` prefix. `~v, ~clientNonce, ~sessionKey` are fresh variables used for rand values, nonces and keys.

## Functions

Functions have a name and an arity (which is the expected numbe rof arguments). All function symbols work on the sort `msg` and all their arguments are `msg`, as is their result. An hash function H of arity 1 can be declared as :

```
functions: H/1
```
This is an example of an *abstract* function. They are simply given a name, an arity, and some of their properties may be
subsequently specified via equations and reasoned about using those equations. However, we do not specify a concrete realization of them.

```
senc() is a function with an arity of 2 that encrypts a tuple <m, ~n> with a key k

senc(<m, ~n>, k)
```
The basic assumption of Tamarin over a function is that the function is one-way and bijective. In later topics we will cover how to specify other properties over a function.

## Substitutions

We write 𝑡𝜎 to denote applying the *substitution 𝜎* to
the term 𝑡. This application replaces each variable 𝑥 in 𝑡 by the term 𝑥𝜎.

Consider the variables v and w, the term t = <v, H(<w, v>)>, and the substitution:

𝜎 = { v ↦ <x, $B>, w ↦ H(~nA) }

We then have:

    - w𝜎 = H(~nA)

    - t𝜎 = < <x, $B>, H(<H(~nA), <x, $B>>) >


## Equality

In Tamarin the equal sign is used to denote equality (it's not the assignment). Without equality Tamarin assumes that two terms are equal if and only if they are syntactically identical. This `H(x)` is different from `G(y)` and if x is different from y, then `G(x)` is different from `G(y)`. This interpretation is too strict for some functions (like decription and encryption). We use the following equation to express that decrypting and encryption with the same key results in the original message:

`sdec(senc(m, k), k) = m`

Diffie Hellman has been defined as follows:

```
(xˆy)ˆz = xˆ(yˆz)
DH_neutralˆx = DH_neutral
x*(y*z) = (x*y)*z
xˆ1 = x
x*1 = x
x*y = y*x
x*inv(x) = 1
```

# Facts

Used to model transitions. The state of the global transition system is a multiset (a set in which a member can occur multiple times) of facts. Except for some built-in symbols, facts are user-defined and their meaning is giveng by their use in the specification.

## Examples
For example, consider modeling an initiator process that consists of several steps, like sending a message, receiving a message, and sending a follow-up message. We could use a fact 
`Initiator(ThreadID, 'state_2', AgentID, m)` to represent that
there exists a thread with identifier ThreadID of an agent AgentID that performs the initiator role (encoded as the fact name), and whose thread is at state_2 after receiving a message m. 

Tamarin has several built-in fact symbols:

- K/1: K(t) is used to check whether the adversary can derive the term t

- In/1: In(t) represents that t was received from the (adversary controlled)
network

- Out/1: Out(t) represents that t was sent to the network

- Fr/1: Fr(t) represents that t was freshly generated.
s
Additionally there are the reserved facts KU/1 and KD/1 that are used internally.

## Linear and persistent facts

As we already saw in the protocol example, Tamarin consume facts from the state and produce other facts.

### Linear Facts: 
Are the one who does not start with `!` and these facts can be removed from the state by transitions (such as Initiator)

### Peristent facts
Are tose that start with `!` (eg: `!LongTermKey($A,...)`) and those are never removed from the state. 

Persistent facts are used for long-term keys and long-term storage of data.


# Rules

A transition system is specified by a set of multiset rewriting rules of the form:

`[ L ]--[ A ]->[ R ]`

L,A,R are each multisets of facts. The actions are used by the property specification language to check on those properties. L and R are the premises and facts.

For each rule, we require that all variables that occur in the actions A or in the right-hand side R also occur in the left-hand side L. The only exception is for public variables, namely those prefixed with a dollar sign $, which may occur in A or R only. This requirement ensures that the transition system is well-defined.

## Tamarin built-ins rules:

- **Fresh rule**: 

Used to model the generation of random values from a sufficiently large space.
`Fresh: []--[]->[ Fr(~x)]`

- **Default network model and message deduction**:

Represent messages sent and received over a network that is fully controlled by the adversary (Dolve-Yao model). Sending a message directly adds the message to the adversary knowledge.
```
[ Out(x) ]-->[ !K(x) ]
[ !K(x) ]--[ K(x) ]->[ In(x) ]
```
Those rules dictates that after a message has been sended the message is known to the attacker and that, if an attacker knows a message, then the message can be received from the network.

the adversary can produce fresh values, and can apply any function to the messages that it already knows.

```
[ Fr(~x)]-->[ !K(~x)]
[ ]-->[ !K($x)]
[ !K(x1)...!K(xn)]-->[!K(f(x1,....,xn))], for all function symbols f
```

# Modeling State Machines

The chapter start with the modelling of a small example, a simple challenge-response protocol that is run between a client and a server. When a client runs the protocol, it
generates a fresh symmetric key k and encrypts this together with a tag ’1’ using
the server’s public key pk(S). It then sends the ciphertext to the server. When the
server receives this message, it decrypts the ciphertext using its private key to obtain k, and responds with the key’s hash h(k).

In addition, we will specify a third state machine that predestirbutes keys as a part of the key infrastructure.

## Key Infrastructure

We assume the private keys are randomly generated and that there is an abstract function pk that, given a private key, can compute the associated public key. In this model we assume that both private (Long-term key Ltk) and public keys will not be removed or updated.

```
1 theory SimpleChallengeResponse
2 begin
3
4 functions: pk/1
5
6 rule Register_pk:
7 [ Fr(~ltk) ] --> [ !Ltk($A, ~ltk), !Pk($A, pk(~ltk)), Out(pk(~ltk)) ]
8 
```

Lines 1 and 2 are needed for all models. Lines 4 states that there exist an abstract function pk with arity 1. This is the function that will produce our public key.

The rule `Register_pk` states that each time this rule is istantiated with any public costant for $A, a new private key of the type ~ltk is created (randomly). Further rules, lemmas etc... are not specified. `$A` it's a public identity that is substitued during the creation of the private key. Remember that Ltk and Pk are Facts, thus you should watch them as a labelling for the system, not as a function.

`!Ltk($A, ~ltk)` is used to store the long-term key on the system so that the owner $A can use it multiple times.

`!Pk($A, pk(~ltk))` stores the public key pk(~ltk) so that clients know
which key to use to encrypt messages for the server $A.

`Out()` is used to publish the public key.

We defined the function for assymmetric encryption and decryption as:

```
function: aenc/2, adec/2
equations: adenc(aenc(m, pk(k)), k) = m
```

We can chose to model decryption by honest participants in two ways:

- Explicitly apply adec to inciming messages

- Pattern matching against the encryption


In this example the second method is used, but for more complex protocol the first may be used.

## Servers

As we already said, the server uses a function h to respond with the hash of the key that has been sended by the client:

`functions: h\1`

The server needs to decrypt the messages that it receive, as we already said we will use pattern matching against the encryption of the message. This means that the server can accept only incoming messages that match the pattern.

```
15 rule Server:
16 [ !Ltk($S, ~ltkS)             // look up the private-key
17 , In( aenc{'1', k}pk(~ltkS) ) // receive a request
18 ]
19 --[ AnswerRequest($S, k)
20 ]->
21 [ Out( h(k) ) ]               // return the hash of the key
```

`aenc{'1', k}pk(~ltkS)` is the equivalent for `aenc( <'1', k>, pk(~ltkS) )`.

## Clients

In contrast to the server role, the
client carries out two steps: sending the initial challenge message, and then waiting
until the response is returned to check that it is indeed as expected. We model each of
these steps with a separate rule:

```
23 rule Client_Step_1:
24 [ Fr(~k) // choose fresh key
25 , !Pk($S, pkS) // lookup public-key of server
26 ]
27 -->
28 [ Client_State_1( $S, ~k ) // store server and key for thread
29 , Out( aenc{'1', ~k}pkS ) // send encrypted key to server
30 ]
```

```
2 rule Client_Step_2:
33 [ Client_State_1(S, k) // retrieve thread state
34 , In( h(k) ) // receive hashed session key from network
35 ]
36 --[ SessKeyC( S, k ) ]-> // state that the session key 'k'
37 []                      // was setup with server 'S'
```

The client uses the fact `Client_State_1` to retrieve the thread state, with the key and associated server identity (Remember that facts are the only way to retreive informations, there are not global variables). The action fact `SessKeyC` is used in later lemmas to ensure that the attacker does not know the private key k.
