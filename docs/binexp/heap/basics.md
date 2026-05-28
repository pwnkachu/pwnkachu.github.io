---
title: Heap Understanding
description: This are my personal notes on how the heap work in various glibc implementations.
---

## Overview

The heap is composed of chunks, structures that contains a fixed amount of metadata and a variable (depends on the chunk lenght that is requested as an input to `malloc()` or `calloc()`) amount of user Data. I'll try to explain how the heap works by break down it's mechanism and components into multiple subchapters. One fundamental component of the heap is the *Allocator*

!!! Allocator
    A heap allocator is a system component that manages a program's dynamic memory by reserving chunks of memory (such as malloc in C or Box/Vec in Rust) and reclaiming them later. It functions as a bookkeeper, tracking free space to minimize fragmentation and prevent memory leaks.

# Chunks

A chunk is a portion of memory on the heap, and an user usually interact with it through a pointer that has been returned by malloc. Every chunk must be freed when it's not longer used and the pointer that referred to the previous allocated chunk should be set to null in order to avoid a UAF (Use After Free).

At level code a chunk is always declared as a `malloc_chunk` and it's field are always the same, but their usage depends on the state of the chunk.

```
struct malloc_chunk {
  INTERNAL_SIZE_T      mchunk_prev_size;  // Size of prev chunk if free
  INTERNAL_SIZE_T      mchunk_size;       // Size in bytes + flags
  struct malloc_chunk* fd;                // Forward pointer
  struct malloc_chunk* bk;                // Backward pointer
  struct malloc_chunk* fd_nextsize;       // Forward pointer for large bins
  struct malloc_chunk* bk_nextsize;       // Backward pointer for large bins
};
```

# Top Chunk

The top chunk is the chunk that is located at the end of the heap. The top chunk is where the memory for `malloc()` is allocated from if there are no bin files of the correct size.
The portion of memory that is still present as freed after a malloc() that took the top chunk memory is called the remainder chunk, which is the new top chunk. The top chunk resides at the highest address of the memory (toward the stack).

# Metadata

- Prev. Size: Is valid only if the previous chunk has been freed. Otherwise this 8  byte space (on 64 bit systems) is taken by the previous chunk to write part of it's user data. This is useful for various purposes. In the first case, the previous chunk has been freed, so the allocator (which manages the chunks) can use this field to perform memory coalescing (merging adjacent free chunks into a larger block) much more quickly.

- Size: This is the overall (data + metadata) size of the chunk. It must be a multiple of 8 meaning that the last 3 bits of the size are 0. This allows to save spaces by putting 3 flags into this field (last three bits):

    - A is the NON_MAIN_ARENA flag that is set to 1 if the current chunk is not located into the main arena. I will dig into arenas later.

    - M is the IS_MMAPED flag and is set to 1 if the chunk is allocated via mmap() rather than malloc(). This means that the chunk was directly created by the OS rather than being taken by the heap. When freed this chunk will be not placed into *bins*, instead a munmap() wil be performed to give the memory back to the OS. Those chunks do not have adjacent chunks, therefore they will ignore A,P flags.

    - P is the PREV_INUSE flag which is set when the previous adjacent chunk is in use. The very first chunk allocated has always this bit set to 1 to prevent access to non owned memory. If it's set to 1 it's impossible to retrieve the previous chunk size.

We must distinguish from freed chunks and allocated chunks:

- *An allocated chunk looks like this:*
    ```
        chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Size of previous chunk, if unallocated (P clear)  |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Size of chunk, in bytes                     |A|M|P|
        mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             User data starts here...                          .
            .                                                               .
            .             (malloc_usable_size() bytes)                      .
            .                                                               |
    nextchunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             (size of chunk, but used for application data)    |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Size of next chunk, in bytes                |A|0|1|
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    ```

- *Free chunks are stored in circular doubly-linked lists, and look like this*:
    ```
        chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Size of previous chunk, if unallocated (P clear)  |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        `head:' |             Size of chunk, in bytes                     |A|0|P|
        mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Forward pointer to next chunk in list             |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Back pointer to previous chunk in list            |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Unused space (may be 0 bytes long)                .
            .                                                               .
            .                                                               |
    nextchunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        `foot:' |             Size of chunk, in bytes                           |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Size of next chunk, in bytes                |A|0|0|
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    ```

In order to explain the Fd and Bk pointers we must give an overview of the *bins*.

# Bins

To improve the efficiency of heap memory allocation and release, glibc malloc uses explicit lists to manage freed chunks. An explicit list is a linked list that connects nodes with the same attribute in series to facilitate management. In glibc malloc, these linked lists are called bins. The nodes on the linked lists are freed chunks. We can differentiate bins in how they are used and the min/max size of the chunks that they store.

### Fastbins
Fast bins are used to improve the allocation efficiency of small memory. Before thread local caches (tcache) were introduced (later in the note), fast bins were the first structure to be accessed during memory application and release. 

On a 64-bit system, a fast bin consists of 10 linked lists (10 fastbins), and the maximum chunk size is 160 bytes. 

However, glibc limits the maximum chunk size to 128 bytes by using the global variable `global_max_fast` by default. Therefore, a chunk whose size is *16 to 128* bytes and is a multiple of 16 is classified as a fast chunk by default (Fastbins size goes from 16 bytes to 128 bytes).

Each fast bin maintains a singly linked list as a LIFO structure. To implement the LIFO algorithm, each fast bin element in the fastbins array points to the tail node of the linked list, and the tail node points to the previous node by using its fd pointer.

*Fast bins do not merge free chunks (Not directly)* and *there isn't a fixed max amount of chunks* that a fastbin can take.

![LIFO](images/LIFO.png)

### SmallBins
Small bins handle memory allocations for fixed-size chunks up to 512 bytes. There are 64 small bins that are structured as a circular doubly linked list (fd and bk are used). The access pattern becomes FIFO and this bins is consolidated when there is a free().


### LargeBins
used to manage freed memory chunks that are larger than 512 bytes (on 64-bit systems). There are 63 large bins, and unlike small bins, each bin holds a range of chunk sizes. This structure requires sorting, which makes allocations slightly slower. Large bins are doubly linked, circular lists (both fd and bk used).

### Unsorted Bin

All remainders from chunk splits, as well as all returned chunks,
are first placed here. They are then placed in regular bins after malloc gives them ONE chance to be used before binning. 

Basically, the unsorted chunks acts as a queue, with chunks being placed on it in free (and malloc_consolidate), and taken off (to be either used or placed in bins) in malloc (either used as the malloc response or if not placed into the associated bin).

### Tcache (Thread Local Cache)

Introduced in glibc 2.26 as an efficiency improvement for the allocation into multithreading programs. They avoid the necessity of gaining the global lock to the main aren for operation on small block. It basically acts as a Fast bins for each thread, in order to save time avoiding to wait for the lock to the main arena. 

The tcache is a LIFO data structure that points only to the head of the list (*only fd is used*)

!!! Main Arena Lock
    The main arena is the primary contiguos memory region created by the glibc memory allocator to manage a process's heap. A lock is present as a mutex to prevent race condition and memory collision in multithread applications.

There are 64 bins that goes from 24 byte to 1032 bytes, with a distance of 16 bytes between each bin. Every tcache bin has a max amount of 7 chunks that it can store.
If the tcache of a specific dimension is full, the associated fastbin is used (requires lock).

# WorkFlow

To fully understand heap management, it is essential to trace the exact path taken by the allocator when memory is requested (`malloc`) and when it is returned (`free`). `glibc` employs a "fast path" logic for common operations and a "slow path" for complex cases or internal maintenance.

### `malloc(size)`

When an application calls `malloc(size)`, the allocator executes a strict sequence of steps. The search stops as soon as a suitable chunk is found.

1. **Request Normalization (Alignment & Padding):**
   The allocator never looks for the exact size requested by the user. It adds the necessary space for metadata (minimum 8 or 16 bytes) and rounds the result up to respect CPU alignment (multiples of 16 bytes on 64-bit systems). If a user asks for 12 bytes, `malloc` will look for a 32-byte chunk.
   *If the request is abnormally large (e.g., hundreds of Kilobytes), `malloc` bypasses all bins and directly uses `mmap()` to request memory from the kernel.*

2. **Tcache Search (Fast Path):**
   `malloc` calculates the Tcache index corresponding to the requested size. If the Tcache bin has at least one chunk available, it removes it from the head of the list (LIFO), updates the pointers, and returns it to the user. *No arena lock is acquired.*

3. **Fastbins Search (Fast Path fallback):**
   If the Tcache is empty but the size falls within the Fastbins range (default < 128 bytes), `malloc` checks the corresponding Fastbin. If it finds available chunks, it takes one. Furthermore, to optimize future allocations, it empties the rest of that Fastbin and moves the remaining chunks into the thread's Tcache, then returns the requested chunk to the user.

4. **Small Bins Search:**
   If the size falls into the Small category (and fast paths failed), the allocator checks the exact associated Small Bin. If it contains a chunk, it is unlinked from the doubly linked list (FIFO) and returned.

5. **Pre-emptive Consolidation (Trigger for Large requests):**
   If the request is for a *Large* size (>= 1024 bytes), `malloc` knows it will have to perform a complex search. Before doing so, it invokes `malloc_consolidate()` to empty all Fastbins, merge adjacent fragments, and place them in the Unsorted Bin, hoping to create a block large enough.

6. **Unsorted Bin Sorting (The Main Engine):**
   This is the heart of the algorithm. If the previous steps fail, `malloc` starts scanning the Unsorted Bin from start to finish (FIFO):
   - **Exact match:** If it finds a chunk of the perfect size, it returns it immediately.
   - **Partial match (Splitting):** If it finds a chunk *larger* than needed, it splits it. It returns the requested portion to the user and turns the remainder into a new free chunk, which is put back into the Unsorted Bin.
   - **Sorting:** If an inspected chunk does not fit, **it is not left in the Unsorted Bin**. `malloc` unlinks it and inserts it into the appropriate Small Bin or Large Bin. *This is the only time Small and Large Bins are populated.*

7. **Large Bins Search:**
   If the request was Large, after sorting the Unsorted Bin, `malloc` searches the corresponding Large Bin. Using the `fd_nextsize` and `bk_nextsize` pointers, it skips through the blocks to find the *best fit* (the smallest block that is still large enough). Again, if the found block is too large, it is split, and the remainder goes into the Unsorted Bin.

8. **Extraction from Top Chunk (The Wilderness):**
   If all bins fail, the allocator turns to the Top Chunk (the large block of memory at the top of the heap bordering unmapped space). If the Top Chunk is large enough, it slices a piece off and returns it.

9. **OS Request (`sysmalloc`):**
   If even the Top Chunk is exhausted or too small, `malloc` is forced to ask the kernel for new memory. It makes a *syscall* (`sbrk` to expand the Top Chunk, or `mmap` to allocate a completely new area) and finally serves the request.

---

### `free(ptr)`

When the application calls `free(ptr)`, the goal of `glibc` is to reclaim the block as quickly as possible, merging it with other blocks only if strictly necessary to combat fragmentation.

1. **Validation and Sanity Checks:**
   `free` does not trust the passed pointer blindly. It checks that:
   - The pointer is correctly aligned in memory.
   - The chunk does not exceed the physical limits of the arena.
   - The chunk is not the same one returned by an immediately preceding `free` call (basic *Double Free* check).
   - If the chunk has the `IS_MMAPPED` (M) bit set to 1, `free` ignores the heap and directly calls the `munmap()` syscall to return the memory pages to the kernel.

2. **Insertion into Tcache (Fast Path):**
   If the chunk size is compatible with the Tcache and the corresponding bin is not full (< 7 elements):
   - **Security:** A check is performed on the chunk's `key` to detect complex *Double Free* attempts within the same bin.
   - The `fd` pointer is written into the data space to point to the current head of the list.
   - The chunk becomes the new head of the Tcache (LIFO). *The process ends here.*

3. **Insertion into Fastbins (Fast Path fallback):**
   If the Tcache is full or disabled, but the chunk falls within the fastbin threshold (e.g., < 128 bytes):
   - **Security:** It checks that the chunk at the top of the list is not the same one being freed.
   - The block is inserted at the head of the Fastbin (LIFO).
   - **Crucial:** The `PREV_INUSE` bit of the physically next block in memory is *not* altered. To the memory, this chunk still appears allocated, preventing any coalescence. *The process ends here.*

4. **Coalescence in the Unsorted Bin (Slow Path):**
   If the block is larger than the fastbins, it must be placed into the Unsorted Bin. Before doing so, `free` tries to merge it with its physical neighbors to create a larger block:
   - **Backward Coalesce:** It checks its own `PREV_INUSE` flag. If it is `0`, it means the physically previous block is free. `free` reads the `prev_size` field, jumps back to the header of the previous block, unlinks it from its current bin, and merges the two chunks by combining their sizes.
   - **Forward Coalesce:** It checks the state of the physically next block (by reading the header of the chunk after that to see its `PREV_INUSE`). If the next block is free, it merges it too. If the next block is the **Top Chunk**, the chunk being freed is simply merged directly into the Top Chunk and the operation ends (it is not placed in any bin).

5. **Insertion into the Unsorted Bin:**
   If the chunk resulting from the merges (or the original chunk if no merges occurred) does not border the Top Chunk, it is inserted at the beginning of the Unsorted Bin's doubly linked list (FIFO).
   - The `PREV_INUSE` bit of the next chunk is set to `0`.
   - The `prev_size` field of the next chunk is updated with the new size of this free block.

6. **Heap Trimming (Optional):**
   If, after a coalescence with the Top Chunk, the total size of the Top Chunk exceeds a certain threshold (typically 128 KB), `free` internally calls `systrim()`. This process attempts to return the excess memory at the top of the heap directly to the operating system via `sbrk` with negative values, lowering the memory footprint of the process.
