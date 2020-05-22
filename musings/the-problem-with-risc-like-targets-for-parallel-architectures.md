---
description: >-
  Ponderings on the limitations of mushing parallelism into things like LLVM or
  RISC-V
---

# The Problem with RISC-like Targets for Parallel Architectures

It's obvious that mushing parallelism into an existing sequential assembly language is messy. I ponder the limitations of using RISCV as a target, and some ways to make a reliable and user-friendly Vector-SIMC compiler.

## The Untamed Chimera Problem

An **untamed chimera compiler** informally merges a foreign semantics \(i.e., parallelism\) with a pre-existing one. By informal, I mean that the compiler makes no guarantees that the foreign semantics will be preserved. The [chimera](https://www.hpl.hp.com/techreports/2004/HPL-2004-209.pdf) that resulted from the insertion of threads in C in 2004 exemplified this problem: formally merging these into a single semantics fixed the problem, yielding a **tamed chimera compiler**. \(I think of the chimera from Fullmetal Alchemist, which was a [very sad chimera](https://www.google.com/search?q=fullmetal+alchemist+chimera&rlz=1C1SQJL_enUS849US850&sxsrf=ALeKk03_-LmgqEH-AW1pSvmOvazSBz-1mg:1588707561002&source=lnms&tbm=isch&sa=X&ved=2ahUKEwiX4czuvJ3pAhVLrZ4KHX6cBC8Q_AUoAXoECBYQAw&biw=1536&bih=722#imgrc=cGwGglkDjKSiSM) story\)

* **Unsound Optimizability--** an untamed chimera compiler cannot be trusted: it will attempt optimizations in the context of the host semantics, with no regards to the target semantics. No matter the engineering effort, the theoretical possibility of the compiler breaking a program remains non-zero. 
  * In spirit, Vector-SIMC actually consists of a control-flow graph, **scalar**, and a set of named basic blocks, **vector**, such that **scalar** performs `vissue`s of blocks in **vector**. Due to the foreign nature of `vissue` semantics, `gcc` as a chimera C/Vector-SIMC compiler doesn't understand the role of **vector**, and may very well treat it as dead code; it must therefore be [carefully ](https://app.gitbook.com/@edwinmpeguero/s/hbir/~/drafts/-M6aAKxGXh-VPNl_5cNI/vector-simc-1.0/tricking-the-compiler-gcc-assumptions)coerced into respecting the semantics.  
* **Incomplete Optimizability--** an untamed chimera compiler lacks awareness of the foreign semantics: therefore, it makes no attempts to perform semantic-specific optimizations. 
  * One might imagine optimizations that hoist `vissue'd` code out of a loop, merge contiguous`vissues` , or altogether removing non-effectful `vissues` , while preserving Vector-SIMC semantics: an untamed chimera compiler cannot possibly do this without incredibly roundabout coercion. 

## Implicit Synchronization in an Untamed Chimera Compiler

**Implicit synchronization,** a key feature of Vector-SIMC, statically inserts worst-case delays before vector cache invalidations \(i.e., `remem`\) to prevent deadlocks without need for synchronization overhead.

This static analysis requires that the compiler compute the "instruction time" delay between two vector cores \($$v_1, v_2$$\), the current instruction of one of the cores:$$delta(v_1, v_2, inst)$$.

An implementation of a tamed chimera compiler with this static analysis must:

1. optimizes the program soundly and completely 
2. computes and inserts synchronizing delays before `remem`
3. lowers the program to machine code

On the other hand, an untamed chimera compiler implementation may perform optimizations during step 3, "invisibly" \(at the machine code level\) rendering the synchronization analysis inaccurate. I need to investigate this potential problem; some starting points: the [`gcc` docs](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Volatile), and this [stackoverflow answer](https://stackoverflow.com/questions/41294779/when-will-compilers-optimize-assembly-code-in-c-c-source).

Thus, each edge case in every layer of an untamed chimera compiler stack must be carefully analyzed to guarantee that the observed behavior matches the intended one, correct or otherwise.

