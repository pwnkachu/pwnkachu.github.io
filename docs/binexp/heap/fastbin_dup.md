---
title: Fastbin Dup
description: These are my personal notes on understanding fastbin duplication.
---

# Fastbin Poisoning & Duplication

Fastbin duplication and poisoning are closely related techniques used when we have a Use-After-Free (UAF) or a heap overflow that permits us to modify the metadata of existing free chunks.

While Duplication specifically refers to tricking the allocator by freeing the same chunk twice (usually in an A -> B -> A pattern), we can use our memory corruption primitive to overwrite the fd pointer of a chunk sitting in a fastbin. The goal is to point this fd to an arbitrary memory location, forcing a subsequent malloc() to return a chunk exactly where we want it.

## Fastbin Size Check

The main problem is that glibc performs a fastbin size sanity check on every chunk being retrieved.

When you request a chunk, the allocator looks at the chunk it is about to hand over (the one at our forged fd address) and checks its size metadata. If the size field of our fake target chunk does not fall into the exact same size class as the fastbin we are pulling from, glibc throws a malloc(): memory corruption (fast) error and crashes the program.
Bypass Strategy 1: The Misaligned Fake Chunk

To bypass this mechanism directly, we need to forge an fd that points to an address where the existing bytes in memory coincidentally look like a valid size field.

- Misalignment: Chunks don't strictly need to be perfectly 8-byte or 16-byte aligned when coming out of a fastbin. We can shift our target pointer back and forth by a few bytes until we frame a byte that matches our fastbin size.

A classic example is targeting __malloc_hook. In older glibc versions, the memory near this hook is populated with libc addresses starting with 0x7f. If we misalign our fake chunk pointer slightly backwards, we can read that 0x7f byte as our fake chunk's size field.

Because 0x7f translates to a size of 0x70 (ignoring the flag bits), we can easily hijack the hook by requesting an allocation of 0x68 bytes (pwndbg `find-fake-fast &addr` command).

## Chunk Overlapping

If we cannot find a valid fake size near our target, we can pivot our strategy entirely by using chunk overlapping. Instead of attacking the fastbin directly, we use it to build a stronger primitive.

We use our UAF or overflow to overwrite the size field of an adjacent, currently allocated chunk. For example, we corrupt a chunk's size from 0x30 to 0x110.

When we eventually free() this corrupted chunk, the allocator looks at our forged massive size. Because it is no longer small enough to fit in a fastbin, it bypasses the fastbin size checks completely and is tossed directly into the Unsorted Bin.

We now have a massive free chunk in the Unsorted Bin that physically overlaps with other chunks that might still be actively in use. This grants us a primitive to leak libc addresses (via the Unsorted Bin's main_arena pointers) or crush adjacent chunk metadata.