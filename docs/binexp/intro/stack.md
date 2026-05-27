---
title: Understanding the Stack 
description: A deep dive into the Stack.
---

# The x86 and x64 Stack on UNIX Systems: Structure, Mechanisms, and Examples

This document serves as a guide and tutorial on the stack in x86 (32-bit) and x64 (64-bit) architectures within UNIX environments (Linux, macOS, BSD).

---

## What is the Stack and How it Works

In the memory model of a UNIX process, the stack is a linear memory region managed with a **LIFO** (Last In, First Out) logic. It is mainly used to:

* Store local variables of functions.

* Save the return address to allow the program to resume execution after a function finishes.

* Pass arguments to functions (especially in x86).

* Save the state of registers (context) before a function call.

### Growth Direction
A fundamental characteristic of the stack on x86/x64 architectures is that it **grows downwards**, meaning towards lower memory addresses.

* When data is pushed onto the stack (`PUSH`), the stack pointer decreases.

* When data is popped from the stack (`POP`), the stack pointer increases.

---

## Key Stack Registers

| x86 Register (32-bit) | x64 Register (64-bit) | Description |
| :--- | :--- | :--- |
| **ESP** (Stack Pointer) | **RSP** (Stack Pointer) | Always points to the current top of the stack (the lowest occupied address). |
| **EBP** (Base Pointer) | **RBP** (Base Pointer) | Acts as the *Frame Pointer*. It points to the base of the current Stack Frame, providing a fixed reference point to access parameters and local variables. |

---

## The Stack Frame

Every time a function is invoked, an isolated block is created on the stack called a **Stack Frame**. This ensures that each function has its own private space for variables and parameters, also enabling recursion.

### Typical Stack Frame Structure (Conceptual View)

``` text
High Addresses (Start of the stack)
+-----------------------------+
| Function Arguments          |  
+-----------------------------+
| Return Address (RET)        |  
+-----------------------------+
| Saved old EBP/RBP           |  <-- Current EBP/RBP
+-----------------------------+
| Local Variables             |  
| and Saved Registers         |  
+-----------------------------+  <-- Current ESP/RSP (Top of the stack)
Low Addresses (Stack growth direction)
```

### Function Prologue and Epilogue

In Assembly, opening and closing a stack frame follows standard patterns.

- Prologue:

    ```
    push ebp        ; Save old base pointer
    mov ebp, esp    ; Set new base pointer
    sub esp, 0x10
    ```
    The prologue works as follows, the ebp is pushed into the stack, in this way the old ebp value is now at the top of the stack. The second instruction is used to move the Base Pointer to the stack pointer, defining the start of the new frame. After that 32 bytes are allocated for local variables.

- Epilogue:

    ```
    mov esp, ebp    
    pop ebp         
    ret
    ```
    Local variables are lost by moving the stack pointer to the start of the frame, after that the previous frame base pointer is retrieved and the return pointer is used to start the execution from previous instruction.

### Crucial Differences: x86 vs x64 on UNIX

UNIX systems follow specific conventions called Application Binary Interface (ABI). For x86, the cdecl convention (System V i386 ABI) is commonly used, while for x64, the System V AMD64 ABI is followed.
1. Parameter Passing

    x86 (32-bit): All parameters are passed via the stack, pushed in reverse order (right to left). The caller is responsible for cleaning up the stack after the call.

    x64 (64-bit): The first 6 integer or pointer arguments are passed directly in dedicated registers to optimize performance. The register order is: RDI, RSI, RDX, RCX, R8, R9. If there are more than 6 arguments, the remaining ones are passed via the stack.

2. Stack Alignment

    x86: The stack generally requires 4-byte alignment.

    x64: The System V ABI requires the stack to be 16-byte aligned right before executing a CALL instruction.

3. Red Zone (x64 only)

    In x64, there is a 128-byte region below RSP called the Red Zone. This zone is protected and is not overwritten by OS signals or interrupts. Leaf functions can use this space for temporary local data without modifying RSP.

4. Frame Pointer Omission (FPO)

    In x64, using RBP as a frame pointer is optional. Modern compilers often use RBP as a general-purpose register, tracking local variables directly via static offsets from RSP.

### Practical Example: C, x86, and x64 Comparison

Let's look at this simple C source code:
```
int sum(int a, int b) {
    int result = a + b;
    return result;
}

int main() {
    int x = sum(5, 10);
    return x;
}
```

#### Compiled x86

```
.intel_syntax noprefix
.global sum
sum:
    push ebp            
    mov ebp, esp        
    sub esp, 4          
    
    mov edx, [ebp + 8]  ; Load 'a'
    mov eax, [ebp + 12] ; Load 'b'
    add eax, edx        
    mov [ebp - 4], eax  
    
    mov eax, [ebp - 4]  
    leave               
    ret                 

.global main
main:
    push ebp
    mov ebp, esp
    
    push 10             ; Second arg
    push 5              ; First arg
    call sum        
    add esp, 8          ; Clean stack
    
    leave
    ret
```

#### Compiled x64

```
.intel_syntax noprefix
.global sum
sum:
    push rbp            
    mov rbp, rsp        
    
    mov [rbp - 8], edi  
    mov [rbp - 12], esi 
    
    mov edx, [rbp - 8]  
    mov eax, [rbp - 12] 
    add eax, edx        
    mov [rbp - 4], eax  
    
    mov eax, [rbp - 4]  
    pop rbp             
    ret                 

.global main
main:
    push rbp
    mov rbp, rsp
    sub rsp, 16         
    
    mov esi, 10         ; Second arg
    mov edi, 5          ; First arg
    call sum        
    
    mov [rbp - 4], eax  
    mov eax, [rbp - 4]  
    
    leave
    ret
```