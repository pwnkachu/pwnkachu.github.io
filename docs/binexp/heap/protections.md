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

- **Heap Leak:**  Must first achieve a heap leak. If an attacker reads the `fd` pointer of the last chunk in a tcache list (which points to `NULL` or $0$), the masked value in memory directly reveals the shift: $(Chunk\_Address \gg 12)$.

- **De-obfuscation:** With the shifted chunk address known, One can reverse the XOR operation mathematically to decode valid pointers or forge new masked pointers pointing to arbitrary memory locations.

---

## Tcache Double Free Protection (Key)
**Introduced:** glibc 2.29
**Location:** Tcache bins (`tcache_entry` structure).

### Mechanism
To prevent a chunk from being freed twice into the same tcache bin, glibc places a `key` field in the `tcache_entry` structure (which overlays the user data region of the chunk). When a chunk is freed, `chunk->key` is set to the memory address of the thread's `tcache_perthread_struct`. If the allocator detects this exact key during a subsequent `free()`, it traverses the specific tcache list to verify if the chunk is already present, aborting if it is.

### Bypass Strategy

- **Key Corruption (UAF):** The most direct bypass utilizes a Use-After-Free (UAF) vulnerability. After the first `free()`, use the dangling pointer to overwrite the `key` field with garbage data (e.g., $0$). The subsequent `free()` will not trigger the list traversal, allowing the double free.

- **Bin Eviction:** Allocate and free other chunks to fill the target tcache bin (reaching the 7-chunk limit). Once full, the target chunk will be freed into a Fastbin or Unsorted Bin, bypassing the tcache-specific checks.

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

- **Pointer Alignment:** Sets $fd = Target\_Address - 0x18$ (on 64-bit architecture) and $bk = Target\_Address - 0x10$.

- **Execution:** When the unlink operation evaluates the state, the integrity checks pass because the calculated offsets at the target address point back to the fake chunk. This results in the target pointer being overwritten, granting an arbitrary write primitive.

---

## FastBins Size check

### Mechanism 

Before removing a chunk from a fastbin glibc will execute a check on the size of the chunk. If the chunk that is going to be returned has a size that is not equivalent to the size of the bin a crash will be caused. This mitigate a lot of fd poisoning techniques on fastbins due to the fact that Arbitrary crafted chunks will often cause a crash due to bad size field.

```
if (__builtin_expect (fastbin_index (chunksize (victim)) != idx, 0))
            {
              errstr = "malloc(): memory corruption (fast)";
            errout:
              malloc_printerr (check_action, errstr, chunk2mem (victim), av);
              return NULL;
            }
```

### Bypass Strategy

In order to bypass this strategy, we should jump to an address which contain a valide size (a full 8 byte size_t) that is large as enough to fit the chunk that we want to hijack. It is notable to say that the chunks don't need to be aligned at 16 bytes. Also if we are trying to jump to libc, we shoul try to look to the size `0x70` as it will accept libc pointers next to a zero in memory.

Common place in which we can jump are `__malloc_hook, _IO_2_1_stdout, __dso_handle`. I will not list any techniques that's based on malloc_hooks, but if interested one can visit: 
[baby_heap](https://github.com/hoppersroppers/nightmare/blob/master/modules/7-Heap/28-fastbin_attack/0ctf_babyheap/readme.md)
that uses malloc hook with a one_gadget.

We can also perform a [fastbin poison chain attacks](https://hazyclimb.dev/posts/fastbin-poison-chains/)