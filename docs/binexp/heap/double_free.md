---
title: Double Free
description: These are my personal notes on understanding and weaponizing double free vulnerabilities in the glibc heap.
---

This document breaks down the classic double free vulnerability, how it completely breaks the glibc heap allocator's internal state, and the methodology used to turn that corruption into an arbitrary memory allocation primitive.

# Core Concept

A Double Free happens when a program frees the same heap chunk twice without reallocating it in between. This usually comes from bad program logic or dangling pointers (a Use-After-Free scenario) where the developer forgets to null out a pointer after calling free().

When you free a chunk, the allocator assumes it's valid and inserts it into a free list (like a Fastbin or a Tcache bin). If you free it again, the allocator might just blindly insert it again, creating a cycle or deeply corrupting the linked list of free chunks.

Let's focus on singly-linked (Tcache and fastbins) lists, as they are the primary and easiest targets for this class of bugs.

Normally, if you free chunk A, and then chunk B, the allocator's internal bin looks like this:
HEAD -> B -> A -> NULL

If we can trick the allocator into accepting chunk A again (often by freeing in an A -> B -> A pattern to bypass older top-of-bin checks), the list gets tangled up:
HEAD -> A -> B -> A

Because A points to B, and B's forward pointer points back to A, the allocator's internal linked list is now circular. We've created an infinite loop inside the heap.

# Exploit 

Once we have that circular loop, we basically own the heap's allocation mechanisms. Here is the standard step-by-step to turn this corrupted list into an arbitrary write:

- Trigger the Double Free: Free the chunks in the A -> B -> A pattern to create the cycle.

- First Allocation (The Overlap): Call malloc() for the target size. The allocator pops the head of the bin and gives you chunk A. The bin now looks like: HEAD -> B -> A.

??? Why
    Even though you now "own" chunk A as an allocated chunk and can write user data into it, the allocator still thinks chunk A is in the free list (because B points to it).

- Pointer Hijacking: You write your target address (e.g., __malloc_hook, a saved RIP on the stack, or a GOT entry) into the first 8 bytes of your allocated chunk A. Because chunk A is simultaneously treated as a free chunk by the allocator, you just overwrote its fd (forward) pointer.

- Second Allocation: You call malloc() again. The allocator gives you chunk B. The bin updates to: HEAD -> A -> Target_Address.

- Third Allocation: You call malloc() again. You get chunk A (again). The bin updates its head to point to your forged pointer: HEAD -> Target_Address.

- The Payoff: You call malloc() one last time. The allocator follows the corrupted fd pointer and returns a chunk directly at your Target_Address. You now have an overlapping chunk on arbitrary memory. Write your payload, and you have code execution.

# Bypassing Double Free Mitigations

As glibc evolved, developers added checks to kill simple double frees, requiring us to get a bit more creative.

## Fastbin Top-of-Chunk Check (glibc < 2.26)

Older glibc versions implemented a fast check to see if the chunk currently being freed was exactly the same as the chunk sitting at the very top of the fastbin.

Bypass: The A -> B -> A trick defeats this entirely. When A is freed the second time, B is at the top of the bin, so the check passes.

## Tcache Double Free Protection (glibc >= 2.29)

As noted in the general heap security mechanisms, tcache now adds a key field over the user data and traverses the bin to check if the chunk is already there.

Bypass: You need a Use-After-Free (UAF) vulnerability. You free chunk A, use the dangling pointer to overwrite the key field with $0$, and then free chunk A again. Since the key is zeroed out, the allocator doesn't realize it's a tcache entry, skips the traversal, and allows the double free. Apart from that, one can always full the tcache bin in order to put chunks into fastbins of the same size. This is useful because fastbins does not have the key field protection.

## Example Challenge