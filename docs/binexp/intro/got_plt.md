---
title: Dynamic Linking
description: In thin note i will explain how Dynamic (and static) linking works in Unix systems.
---

## Overview

Most of modern programs have some type of dependendencies that they will need to load at runtime in orderd to access those dependencies. One executable con either *Statically* link those dependencies at build time or *Dynamically* link those at runtime, using a little more time to solve the dependency during the program execution. Both have their pro and trade-off [Static vs Dynamic](https://www.reddit.com/r/linux/comments/i6ucyr/a_discussion_on_static_vs_dynamic_linking/) but in this note we'll analyze what are the direct changes into the ELF.



### Static Linking

Static Linking will resolve all the dependencies in the linking phase. This means that once all object files are created the linker will search the needed dependencies ad include them into the resulting executable files. In order to provide those dependencies, the actual machine code of the libraries is copied into the final executable files. Obviously this result into a larger file. Those functions that are statically linked will be found at a fixed (static) location relative to the rest of the application code.

### Dynamic Linking

The linker will adds placeholder into a section that will later be used to resolve the dependencies at runtime. This means that the files is smaller and that the function will be resolved only if they are actually called at runtime. If our dynamically linked file calls a function foo() at runtime, the dynamic linker will need to check the `.dynsym` section that contains the symbols needed for program execution. This includes the functions your program calls from shared libraries (like printf) and the symbols your binary exports to others.

An example of .dynsym from the buffer overflow code:

```
Symbol table '.dynsym' contains 10 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _[...]@GLIBC_2.34 (2)
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND [...]@GLIBC_2.2.5 (3)
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND [...]@GLIBC_2.2.5 (3)
     4: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     5: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND gets@GLIBC_2.2.5 (3)
     6: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND [...]@GLIBC_2.2.5 (3)
     7: 0000000000403040     8 OBJECT  GLOBAL DEFAULT   25 [...]@GLIBC_2.2.5 (3)
     8: 0000000000403050     8 OBJECT  GLOBAL DEFAULT   25 stdin@GLIBC_2.2.5 (3)
     9: 0000000000403060     8 OBJECT  GLOBAL DEFAULT   25 [...]@GLIBC_2.2.5 (3)

```
The .synsym contains either Elf32 or Elf64 symbols describing an imported symbol and referencing the offset of that symbol (as gets@GLIBC_2.2.5) string in the .`dynstr` section

# GOT and PLT

To perform lazy binding (runtime lookup of symbols that are needed by the executable) the *Procedure Linkage Table* and *Global Offset Table* are used. 

In order to lookup a function `foo()` in library lib.so the dynamic linker (ld.so or ld-linux.so) needs a lookup function for searching the current address of `foo()`. The lookup function is `_dl_runtime_resolve()` that resides in the dynamic linker. This functions needs the module in which the binding occurs (eg: main) and the symbol of the function `foo()`. 

Is important to notice that if a binary is statically linked the GOT will be read only (already filled with dependencies addresses)

## Logic

If the is invoked for the first time the GOT will not contain the address of the needed function. The lookup will obtain the address of the function based on the module ID and the ID of the invoked function (pushed by the PLT) and fill the corresponding entry of the GOT. Next time that function is called, the GOT will be used to directly jump at the address of the function

## PLT

PLT is executable and readable. Looking at the code with a debugger we will se a call to an external function as the following one:

```
call   foo@plt 
```
The call instruction will jump to the PLT entry of `foo`

```
foo@plt:
    jmp *(foo@got)
    push n
    push moduleID
    jmp _dl_runtime_resolve
```
This PLT is going to jump to the entry of `foo` in GOT. If the entry has te correct function address then the function is called. If this function is invoked for the first time the content of the GOT entry is the address of the second instruction of the PLT `push n`. The PLT is going to push the `foo` ID into the stack and the ID of the module, after that a jump to the resolver is performed.

```
Disassembly of section .plt:

0000000000400360 <printf@plt-0x10>:
  400360:	ff 35 8a 2c 00 00    	push   0x2c8a(%rip)        # 402ff0 <_GLOBAL_OFFSET_TABLE_+0x8>
  400366:	ff 25 8c 2c 00 00    	jmp    *0x2c8c(%rip)        # 402ff8 <_GLOBAL_OFFSET_TABLE_+0x10>
  40036c:	0f 1f 40 00          	nopl   0x0(%rax)

0000000000400370 <printf@plt>:
  400370:	ff 25 8a 2c 00 00    	jmp    *0x2c8a(%rip)        # 403000 <printf@GLIBC_2.2.5>
  400376:	68 00 00 00 00       	push   $0x0
  40037b:	e9 e0 ff ff ff       	jmp    400360 <_init+0x24>

0000000000400380 <execve@plt>:
  400380:	ff 25 82 2c 00 00    	jmp    *0x2c82(%rip)        # 403008 <execve@GLIBC_2.2.5>
  400386:	68 01 00 00 00       	push   $0x1
  40038b:	e9 d0 ff ff ff       	jmp    400360 <_init+0x24>

```
PLT[0] is the code that will call the linker. Used by all PLT entries to avoid duplication of code. We can see that PLT[0] is entitled to push into the stack the pointer to the link_map structure and then jumps to GOT[2] that is the resolver address.

## GOT

The first 3 entry of the GOT are special entries:

-   GOT[0] contains the address of the `.dynamic` section, which 
    contains crucial informations for the dynamic linker such as the pointers to `.dynsym` and `.dynstr` and the list of the external libraries that the binary depend on. This is inserted by the static linker at compile time.

-   GOT[1] points at the `link_map` structure that is used by 
    the linker to navigate the shared libraries that are present in memory. This is a Glibc structure.

-   GOT[2] is the address of `_dl_runtime_resolve()`

GOT is writable and readable.

```
Global Offset Table '.got' contains 2 entries:
 Index:    Address       Reloc         Sym. Name + Addend/Value
     0: 000000402fd8 R_X86_64_GLOB_DAT __libc_start_main@GLIBC_2.34 + 0
     1: 000000402fe0 R_X86_64_GLOB_DAT __gmon_start__ + 0

Global Offset Table '.got.plt' contains 7 entries:
 Index:    Address       Reloc         Sym. Name + Addend/Value
     0: 000000402fe8                   402e08
     1: 000000402ff0                   0
     2: 000000402ff8                   0
     3: 000000403000 R_X86_64_JUMP_SLO printf@GLIBC_2.2.5 + 0
     4: 000000403008 R_X86_64_JUMP_SLO execve@GLIBC_2.2.5 + 0
     5: 000000403010 R_X86_64_JUMP_SLO gets@GLIBC_2.2.5 + 0
     6: 000000403018 R_X86_64_JUMP_SLO setvbuf@GLIBC_2.2.5 + 0
```

We can see that the GOT is divided into two sections. .got is used for functions that needs to be resolved immediatly at the program startup. `R_X86_64_GLOB_DAT` entries are always resolver at the startup by the ld.

`.got.plt` contains those functions that will be resolved at runtime.


### link_map
As we said this structure is used to reference the external libraries that are needed by the binary. The structure is a list used to reference useful information for every needed library and the first entry of the list is the current executable (the address was obtained by printing GOT[1] with pwndbg):

```
p *(struct link_map *) 0x7ffff7ffe2e0

$1 = {
  l_addr = 0,
  l_name = 0x7ffff7ffe8b8 "",
  l_ld = 0x402e08,
  l_next = 0x7ffff7ffe8c0,
  l_prev = 0x0,
  l_real = 0x7ffff7ffe2e0,
  l_ns = 0,
  l_libname = 0x7ffff7ffe8a0,
  l_info = {0x0, 0x402e08, 0x402ee8, 0x402ed8, 0x0, 0x402e88, 0x402e98, 0x402f18, 0x402f28, 0x402f38, 0x402ea8, 0x402eb8, 
    0x402e18, 0x402e28, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x402ef8, 0x402ec8, 0x0, 0x402f08, 0x0, 0x402e38, 0x402e58, 0x402e48, 
    0x402e68, 0x0 <repeats 13 times>, 0x402f58, 0x402f48, 0x0 <repeats 13 times>, 0x402f68, 0x0 <repeats 25 times>, 0x402e78},
  l_phdr = 0x400040,
  l_entry = 4195248,
  l_phnum = 13,
  l_ldnum = 0,
  l_searchlist = {
    r_list = 0x7ffff7fbb620,
    r_nlist = 3
  },
  l_symbolic_searchlist = {
    r_list = 0x7ffff7ffe898,
    r_nlist = 0
  },
  l_loader = 0x0,
  l_versions = 0x7ffff7fbb640,
  l_nversions = 4,
  l_nbuckets = 3,
  l_gnu_bitmask_idxbits = 0,
  l_gnu_shift = 6,
  l_gnu_bitmask = 0x401030,
  {
    l_gnu_buckets = 0x401038,
    l_chain = 0x401038
  },
  {
    l_gnu_chain_zero = 0x401028,
    l_buckets = 0x401028
  },
  l_direct_opencount = 1,
  l_type = lt_executable,
  l_dt_relr_ref = 0,
  l_relocated = 1,
  l_init_called = 1,
  l_global = 1,
  l_reserved = 0,
  l_main_map = 0,
  l_visited = 1,
  l_map_used = 0,
  l_map_done = 0,
  l_phdr_allocated = 0,
  l_soname_added = 0,
  l_faked = 0,
  l_need_tls_init = 0,
  l_auditing = 0,
  l_audit_any_plt = 0,
  l_removed = 0,
  l_contiguous = 1,
  l_free_initfini = 0,
  l_ld_readonly = 0,
  l_find_object_processed = 1,
  l_tls_in_slotinfo = 0,
  l_nodelete_active = false,
  l_nodelete_pending = false,
  l_has_jump_slot_reloc = false,
  l_property = lc_property_valid,
  l_x86_feature_1_and = 0,
  l_x86_isa_1_needed = 1,
  l_1_needed = 0,
  l_rpath_dirs = {
    dirs = 0xffffffffffffffff,
    malloced = 0
  },
  l_reloc_result = 0x0,
  l_versyms = 0x4011b2,
  l_origin = 0x0,
  l_map_start = 4194304,
  l_map_end = 4206704,
  l_scope_mem = {0x7ffff7ffe5d8, 0x0, 0x0, 0x0},
  l_scope_max = 4,
  l_scope = 0x7ffff7ffe680,
  l_local_scope = {0x7ffff7ffe5d8, 0x0},
  l_file_id = {
    dev = 0,
    ino = 0
  },
  l_runpath_dirs = {
    dirs = 0xffffffffffffffff,
    malloced = 0
  },
  l_initfini = 0x7ffff7fbb600,
  l_reldeps = 0x0,
  l_reldepsmax = 0,
  l_used = 1,
  l_feature_1 = 0,
  l_flags_1 = 0,
  l_flags = 0,
  l_idx = 0,
  l_mach = {
    plt = 0,
    gotplt = 0,
    tlsdesc_table = 0x0
  },
  l_lookup_cache = {
    sym = 0x401128,
    type_class = 2,
    value = 0x7ffff7fbb0c0,
    ret = 0x7ffff7f30990
  },
  l_tls_initimage = 0x0,
  l_tls_initimage_size = 0,
  l_tls_blocksize = 0,
  l_tls_align = 0,
  l_tls_firstbyte_offset = 0,
  l_tls_offset = 0,
  l_tls_modid = 0,
  l_tls_dtor_count = 0,
  l_relro_addr = 4206072,
  l_relro_size = 520,
  l_serial = 0
}
```
This is a very large struct thus i will describe only some fields.
- `l_addr` is at 0 because the executable has been compiled     
    without PIE. It normally contains the offset from the base address.

- `l_name` contains an empty string because this is the  
       executable file, it normally contains the name of the library.

- `l_ld` is the pointer to the `.dynamic` section (GOT[0])

- `l_prev` and `l_next` are the pointers of the list.

- `l_info` is used by the linker to store all the pointers taken by the `.dynamic` section. Used to avoi recompilation of the ELF.

- `l_map_start` and `l_map_end` contains the start and the end of the virtual memory of the program.

### .dynamic

GOT[0] entry that is used by the loader to know the needed libraries (NEEDED) and other informations, such as the string tab address and the got address (PLTGOT). This is readed at the startup of the executable to initialize the `link_map` struct at the `l_info` field.

```
readelf -d challenge

Dynamic section at offset 0x1e08 contains 24 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x000000000000000c (INIT)               0x40033c
 0x000000000000000d (FINI)               0x400574
 0x0000000000000019 (INIT_ARRAY)         0x402df8
 0x000000000000001b (INIT_ARRAYSZ)       8 (bytes)
 0x000000000000001a (FINI_ARRAY)         0x402e00
 0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)
 0x000000006ffffef5 (GNU_HASH)           0x401020
 0x0000000000000005 (STRTAB)             0x401140
 0x0000000000000006 (SYMTAB)             0x401050
 0x000000000000000a (STRSZ)              114 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000015 (DEBUG)              0x0
 0x0000000000000003 (PLTGOT)             0x402fe8
 0x0000000000000002 (PLTRELSZ)           96 (bytes)
 0x0000000000000014 (PLTREL)             RELA
 0x0000000000000017 (JMPREL)             0x401270
 0x0000000000000007 (RELA)               0x4011f8
 0x0000000000000008 (RELASZ)             120 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000006ffffffe (VERNEED)            0x4011c8
 0x000000006fffffff (VERNEEDNUM)         1
 0x000000006ffffff0 (VERSYM)             0x4011b2
 0x0000000000000000 (NULL)               0x0
```


Sources:

- [General](https://www.openeuler.org/en/blog/lijiajie128/2020-11-10-%E5%8A%A8%E6%80%81%E9%93%BE%E6%8E%A5%E4%B8%AD%E7%9A%84PLT%E4%B8%8EGOT)

- [GOT](https://maskray.me/blog/2021-08-29-all-about-global-offset-table)