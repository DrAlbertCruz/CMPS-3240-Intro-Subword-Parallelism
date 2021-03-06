# CMPS-3240-Intro-Subword-Parallelism
An introduction to Intel's SSE instruction set at the assembly level

# Introduction

## Objectives

* Learn how to implement SSE instructions at the assembly level

## Prerequisites

* Understand System V calling convention in x86
* Understand the concept of SIMD

## Requirements

### General

* Familiarize yourself with Streaming SIMD Extensions (SSE) instructions<sup>2</sup> `movupd` and arithmetic instructions

### Software

This lab requires `gcc`, `make`, and `git`.

### Hardware

This lab requires a CPU that has the SSE instruction set. It was originally introduced with the Pentium III so if you're using a PC made after 1999 you most likely have it.<sup>3</sup> You can check with the following command in linux:

```shell
$ cat /proc/cpuinfo | grep "sse"
flags		: ... sse sse2 ...
```

and look for `sse` and `sse2` under the `flags` field. Again, you'll most likely have it unless you're using a Mac that has a PowerPC processor or you're doing this on a system-on-chip with an ARM processor (Raspberry Pi). In those scenarios you'll have a bigger problem: you don't have an Intel or AMD x86 processor.

### Compatability

| Linux | Mac | Windows |
| :--- | :--- | :--- |
| Yes | Yes<sup>*‡</sup> | Untested<sup>*</sup> |

<sup>*</sup>This lab should work across all environments, assuming you set up `gcc` correctly. The lab manuel is written for `odin.cs.csubak.edu`, but it should not be too much of a stretch in other environments. The only concern is if you're using Windows, you need to use `movapd` instead of `movupd` (notice the subtle difference with the `a` and the `u`. Windows stores SIMD arrays in a particular way (aligned) that is different from POSIX OS (unaligned).

<sup>‡</sup>Mac has the additional difference that it uses `clang` rather than `gcc`, so the lab manual's background analysis of the code generated by the compiler might not apply.

## Background

The topic of today's lab is the optimization of an element-wise vector multiplication. For this lab I will call the operation *DEWVM* but this is not a standard notation. It takes two vectors of identical size and steps through the arrays, multiplying it element by element and storing the result in a third array:

```c
int dewvm( int N, double *x, double *y, double *result ) {
    for( int i = 0; i < N; i++ )
        result[i] = x[i] * y[i];

    return 0;
}
```

which seems simple enough. Note that these are double precision, which is 64 bits large. If you run benchmark this operation three times on `odin.cs.csubak.edu` you get the following results:

```shell
$ for i in {1..3}; do time ./bench_dewvm.out; done;

real    0m0.877s
user    0m0.460s
sys     0m0.416s

real    0m0.823s
user    0m0.452s
sys     0m0.368s

real    0m0.881s
user    0m0.484s
sys     0m0.396s
```

Recall from lecture that there are a type of instructions called SIMD, which stands for single instruction multiple datapath (SIMD). The SIMD instruction set we will be using today is called SSE, and it was introduced with the Pentium 4. Essentially, when iterating an arithmetic operation over an array, rather than carry it out one index at a time, we can:

1. Grab N values from the arrays and place them into a single, large register that can fit them all. The oversized registers in x86 are called *multimedia registers* (MM), so called because they were originally introduced to give better performance and fidelity for--as you might have guessed, multimedia programs. The process of stuffing the MM registers with many variables is called *packing*. An MM register is *packed* if it contains many smaller values inside of it.
1. Carry out a single arithmetic operation, that will be carried out on all the values in the oversized register. So, rather than carry out N arithmetic operations on N data points, we carry out only 1 arithmetic operation that automatically operates on N data points. 
1. *Unpack* the MM register by placing the items back in order into memory.

This introduces the constraint that the array size must be a multiple of N, but potentially improves execution time by a factor of 1/N. Though, not quite, as there is some overhead introduced by packing and unpacking the arrays. I get the following result on `odin.cs.csubak.edu` when I alter the code to make use of SSE SIMD instructions:

```shell
$ for i in {1..3}; do time ./bench_dewvm.out; done;

real    0m0.716s
user    0m0.284s
sys     0m0.428s

real    0m0.657s
user    0m0.324s
sys     0m0.332s

real    0m0.661s
user    0m0.280s
sys     0m0.384s
```

SSE instructions use a 128-bit MM registers. So, we can pack 2 doubles into a single MM register, and we will need to carry out half as many multiplications. In practice, the improved code benches at an average of 296ms from 465ms (remember that you care about user time, not system time). It is faster by a factor of 0.636, so, not exactly 1/2 but pretty close.

### Addressing in x86

For this lab you will need to know advanced memory addressing in x86 GAS. The following example was taken from this reference.<sup>1</sup>

```x86
mov (%ebx), %eax 	/* Load 4 bytes from the memory address in EBX into EAX. */
mov %ebx, var(,1) 	/* Move the contents of EBX into the 4 bytes at memory address var.
(Note, var is a 32-bit constant). */
mov -4(%esi), %eax 	/* Move 4 bytes at memory address ESI + (-4) into EAX. */
mov %cl, (%esi,%eax,1)    	/* Move the contents of CL into the byte at address ESI+EAX. */
mov (%esi,%ebx,4), %edx      	/* Move the 4 bytes of data at address ESI+4*EBX into EDX. */ 
```

Essentially, `(%ebx)` is shorthand for `0(0,%ebx,1)` and empty elements in this notation are implied 0 or 1. The generalized form is `a(b,c,d)` which evaluates to the memory address at `b + d * c + a`. `b` and `c` are generally registers, whereas `a` and `d` are generally literals. Note that within memory addressing, the numbers do not need a `$` prefix. For this lab we will use the `lea` acdress. The scale factor, `d`, must be 1, 2, 4 or 8.

# Approach

The approach for this lab is as follows:

1. Study `myblas.c` to understand whats going on at a high level
1. Study `myblas.s` to understand whats going on at the assembly level
1. Follow the instructions to insert SSE commands
1. Assemble a binary, and compare the performance of the unoptimized vs. optimized code

## Part 1 - Study the code

This lab comes with the following files:

* `myblas.c` - Code for our BLAS library which, for this lab only, is limited to the DEWVM operation. You should study this to get an idea of whats going on with the math, and how the function should operate.
* `myblas.h` - Header containing function definitions for our BLAS library
* `myblas.s` - Some base assembly code that was assembled on `odin.cs.csubak.edu`. 
* `bench_dewvm` - A program to benchmark DEWVM. Run this to get times.
* `test_dewvm` - A program to test DEWVM to make sure the result is correct. Run this to make sure it works properly.
* `makefile` - A makefile that contains the targets you will need for the lab. `all` should be sufficient. If you're doing this in another environment, or want to start over with `myblas.s`, reset it with `make assemble`. Note that `myblas.o` links from `myblas.s`, and not the C code.

Currently the code does something like this:

### Calculate `*(result + i*8)`

First, the compiler does pointer math for `*(result + i*8)`. The compiler just decided to do this first, and if it's not broke don't fix it. So just study it and don't change it. On `odin.cs.csubak.edu` I get the following on line 22:

```x86
    movl    -4(%rbp), %eax      # Get `i`
    cltq                        # Promote `i`
    leaq    0(,%rax,8), %rdx    # RDX <- i * 8
    movq    -48(%rbp), %rax     # RAX <- *result
    addq    %rdx, %rax          # RAX += i*8
```

The first two lines are not pointer math at all. `-4(%rbp)` is where the C variable `i` lives on the stack, and it gets this value from the stack. You might wonder why it does not just load it once and keep it in a register. The compiler in general does not do a good job at optimizing assembly code, and we only have `rax`, `rdx` and `rcx` as scratch registers. If you intend to use others, you must save their original value on the stack and restore them before returning from the function call. `i` was initialized as a 32-bit integer and `cltq` promotes it to a 64-bit value. 

The third line calculates the offset based on `i`. Recall that we need to calculate `result[i]`, and that memory addresses are byte addressable. `double`s are 64 bits, so we need to skip every 8 bytes. Why is it `lea`? The compiler chose to do this. See the background section for what happens here. It is literally `i*8`. You could probably have used an unsigned multiplication in place here, but arithmetic operations are a cummulative sum, and LEA here does not require you to zero our RDX before you use it. Note that `-48(%rbp)` is the location of `*result`, and the last two lines finally carry out the pointer math.

### Calculate `*(x + i*8)` and fetch value

```x86
    movl    -4(%rbp), %edx      # Reshadow `i`
    movslq  %edx, %rdx          # A different way to promote it to 64 bits
    leaq    0(,%rdx,8), %rcx    # RCX <- i*8
    movq    -32(%rbp), %rdx     # RDX <- *x
    addq    %rcx, %rdx          # RDX += RCX
    movsd   (%rdx), %xmm1       # XMM1 <- Mem(RDX)
```

The compiler, in a very unoptimized fashion, decided to reshadow `i` and calculate `i*8`. `movslq` is another way to promote a 32-bit integer to a 64-bit integer. This pretty much follows the previous subsection except for the last line, which grabs the memory address at RDX and places it in register.

We have not yet interacted with an x86's MM registers. Depending on how new your processor is, you will have a varying number of, and size of your floating point registers. With the latest processors (AVX-512) you have 32 mm registers. The prefix of the register indicates the size:

* `%zmm0` through `%zmm31`: 512-bit floating point registers. 
* `%ymm0` through `%ymm15`: 256-bit floating point registers. 
* `%xmm0` through `%xmm15`: 128-bit floating point registers. 

SSE operates on 128-bit instructions. You might have noticed that the smallest MM registers are already double-wide, because a double precision number takes 64 bits. Even without SSE, the compiler is using registers intended for SIMD operations. So, currently it looks like this:

| First half | Second half |
| --- | --- |
| Empty | `*(x + i*8)` |

What we need to do here is modify the last command so that instead of fetching just `x[i]`, we fetch both `x[i]` and `x[i+1]`. We do this with the `movupd` command in POSIX or `movapd` in Windows:

```x86
    movupd   (%rdx), %xmm1       # Line 30, previously just movsd
```

That's it. The processor will handle placing the two values `x[i]` and `x[i+1]` into the double-wide register `%xmm1`. From above, we do not need to change the register notation to be larger because the smallest MM register can already accomodate two double precision values. After this operation, the contents of the MM register will look like this:

| First half | Second half |
| --- | --- |
| `*(x + i*16)` | `*(x + i*8)` |

### Calculate `*(y + i*8)` and fetch value

This is similar to the above section. What you need to do is modify line 36 similarly to above.

### Carry out multiplication, store result, increment counter

The final part of the code is on line 45:

```x86
    mulsd   %xmm1, %xmm0    # Multiply two scalars
    movsd   %xmm0, (%rax)   # Store result back into *(result + i*8)
    addl    $1, -4(%rbp)    # i++
```

`mulsd` carries out the multiplication. Note that the `s` implies scalar, so it will multiply the 64-bit value in the first argument and the 64-bit value in the second argument (but only the second halves). `movsd` moves a single 64-bit value into a memory location (only the lower 64-bits of `xmm`, copied into a single 64-bit position in memory). Finally, the last instruction carries out `i++`. Modify it as follows:

```x86
    mulpd   %xmm1, %xmm0    # Multiply two scalars
    movupd   %xmm0, (%rax)   # Store result back into *(result + i*8)
    addl    $2, -4(%rbp)    # i++
```

The `pd` suffix tells the processor that the contents of the MM registers are vectors, not scalars, and it will operate on the first halves and then the second halves. This applies to both the multiplication with `mulpd`, and storing two values into memory with `movupd` rather than just 1. Note that since each operation does twice the work, we need to increment `i+=2` instead of just one. 

## Part 2 - Assemble, test and bench

Carry out the following once you've made the changes above:

* Assemble the files the the `all` target (not the `assemble` target)
* Run `./test_dewvm.out` to make sure it works properly
* Benchmark the process three times with `./bench_dewvm.out`

# Check off

For credit, show your `.s` code and timings to the instructor.

# References

<sup>1</sup>http://flint.cs.yale.edu/cs421/papers/x86-asm/asm.html

<sup>2</sup>https://docs.oracle.com/cd/E26502_01/html/E28388/eojde.html

<sup>3</sup>https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions

<sup>4</sup>https://en.wikipedia.org/wiki/Advanced_Vector_Extensions#AVX
