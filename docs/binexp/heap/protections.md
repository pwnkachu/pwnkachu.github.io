---
title: Heap Understanding
description: This are my personal notes on how the heap work in various glibc implementations.
---

# Glibc Heap Security Mechanisms

This document outlines the primary security mitigations implemented in the glibc heap allocator, their specific locations within the heap architecture, and the theoretical methodologies required to bypass them.

## Safe Linking
**Introduced:** glibc 2.32
**Location:** Singly-linked lists (Tcache and Fastbins).

### Mechanism
Safe Linking obfuscates the forward pointers (`fd` or `next`) of free chunks to prevent trivial pointer hijacking. It leverages Address Space Layout Randomization (ASLR) entropy. The stored pointer is masked using the chunk's own memory address:
$Masked\_Pointer = (Chunk\_Address \gg 12) \oplus Target\_Pointer$

### Bypass Strategy
Safe Linking is deterministic and relies entirely on the secrecy of the heap base address. 

- **Heap Leak:** An attacker must first achieve a heap leak. If an attacker reads the `fd` pointer of the last chunk in a tcache list (which theoretically points to `NULL` or $0$), the masked value in memory directly reveals the shift: $(Chunk\_Address \gg 12)$.

- **De-obfuscation:** With the shifted chunk address known, the attacker can reverse the XOR operation mathematically to decode valid pointers or forge new masked pointers pointing to arbitrary memory locations.

---

## Tcache Double Free Protection (Key)
**Introduced:** glibc 2.29
**Location:** Tcache bins (`tcache_entry` structure).

### Mechanism
To prevent a chunk from being freed twice into the same tcache bin, glibc places a `key` field in the `tcache_entry` structure (which overlays the user data region of the chunk). When a chunk is freed, `chunk->key` is set to the memory address of the thread's `tcache_perthread_struct`. If the allocator detects this exact key during a subsequent `free()`, it traverses the specific tcache list to verify if the chunk is already present, aborting if it is.

### Bypass Strategy

- **Key Corruption (UAF):** The most direct bypass utilizes a Use-After-Free (UAF) vulnerability. After the first `free()`, the attacker uses the dangling pointer to overwrite the `key` field with garbage data (e.g., $0$). The subsequent `free()` will not trigger the list traversal, allowing the double free.

- **Bin Eviction:** An attacker can allocate and free other chunks to fill the target tcache bin (reaching the 7-chunk limit). Once full, the target chunk will be freed into a Fastbin or Unsorted Bin, bypassing the tcache-specific checks.

---

## Safe Unlink
**Introduced:** glibc 2.3.6
**Location:** Doubly-linked lists (Small Bins, Large Bins, Unsorted Bin during immediate or deferred consolidation).

### Mechanism
Before removing a chunk from a doubly-linked list, the `unlink` macro validates the integrity of the adjacent nodes to prevent arbitrary writes during the pointer detachment phase. It enforces the following conditions:
$P \rightarrow fd \rightarrow bk == P$
$P \rightarrow bk \rightarrow fd == P$
If the forward chunk's backward pointer (or the backward chunk's forward pointer) does not point back to the current chunk $P$, the allocator detects corruption and aborts.

### Bypass Strategy
Bypassing the safe unlink check requires a known pointer to the victim chunk (often found in the `.bss` segment, global variables, or the stack).

- **Fake Chunk Forgery:** Craft a fake chunk payload within the victim chunk.

- **Pointer Alignment:** The attacker sets $fd = Target\_Address - 0x18$ (on 64-bit architecture) and $bk = Target\_Address - 0x10$.

- **Execution:** When the unlink operation evaluates the state, the integrity checks pass because the calculated offsets at the target address point back to the fake chunk. This results in the target pointer being overwritten, granting an arbitrary write primitive.

---