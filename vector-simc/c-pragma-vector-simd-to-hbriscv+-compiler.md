---
description: >-
  A description of lotus, a compiler for a C pragma extension for Vector-SIMD
  kernel specification that targets a gem5-simulated HammerBlade architecture
---

# Vector-SIMC to HBRV Compiler

For our first stab at a formal language for a Vector-SIMD kernel, we would like to add special `pragma`s into C, calling this new language **Vector-SIMC** _****_for the lulz. 

Before that, however, we will first contemplate an equivalent `pragma` -less version of **Vector-SIMC**.

Then, we consider **lotus**, a **Vector-SIMC** compiler that targets a Vector-SIMD extension of RISCV, **HBRISCV**, which runs on a `gem5` simulation of _HammerBlade_ developed by Phil Bedoukian.

This version of **lotus** must accomplish the following tasks:

1. Separate **scalar** and **vector** from a high-level Vector-SIMD kernel specification
2. Compile both **scalar** and **vector** into **HBRISCV** via `gcc -S`
3. Glue the two assembly files together to form a **HBRISCV** kernel
4. Compile this **HBRISCV** kernel into an object file via `gcc -c`

## Vector-SIMC

**Vector-SIMC** uses `#define SCALAR`,`#define VECTOR` , and `gcc -E` \(the preprocessing functionality of `gcc`\) to separate two CFGs: **scalar** and **vector**. Respectively, they allow the programmer to explicitly separate _control flow/data movement_ from _compute_.

**scalar** consists of the kernel's control flow structures, with all _control-contiguous blocks_ replaced with a corresponding `vissue` _`block_label`_ instruction. We denote instructions to be **control-contiguous** if they reside at the same level in a control-flow structure \(e.g., both are inside the innermost loop\) and transitively extend this property over basic blocks. The _vissue_ instruction broadcasts the basic block at `block_label` to all _vector cores_. The `block_label` refers to a C-language label consisting of a dummy `asm` name; this name refers to a control-contiguous block in **vector** demarcated by two delimiters of the same name, but post\_fixed with `start` and `end`. Additionally, **scalar** performs _data prefetching_ into vector core memories via the`vprefetch` instruction, and synchronization is accomplished implicitly via a [vector clock](https://en.wikipedia.org/wiki/Vector_clock).

**vector** mirrors the kernel's control-flow structures, but uses only `while` and `if` structures with indeterminate predicates. `asm` "pragmas" \(i.e., comments/BS instructions\) name and demarcate control-contiguous blocks referred to in **scalar**. Control-flow structures are necessary for `gcc` optimizations to accurately respect and exploit control-flow data dependencies.

At the start of a Vector-SIMD kernel, a `vmask` instruction disables all vector core instruction caches \(I-caches\), leaving only the scalar core running. At the end of the kernel, the scalar core executes a `devec` instruction to re-enable vector core I-caches. All cores then wait for each other and finally `return`. 

Note that both **scalar** and **vector** share the`return` control-flow structure, which compiles to two **HBRISCV** steps:

1. a **return block** consisting of a series of stack-manipulations that revert stack registers to their original values 
2. a final jump to the return address `ra`

Correct behavior requires that **scalar/vector** cores perform their respective stack-manipulations before jumping to their corresponding return addresses together. To avoid running a return block on the wrong core, the scalar core must execute the **scalar return block** and send a `vissue` with the **vector** **return block** before calling `devec`. In Vector-SIMC scalar and vector code, we mark **return blocks** by adding an `asm` label immediately before the `return` statement. The compiler assumes that the emitted assembly between this label and the next jump instruction to the return address \(`ra` \) exactly demarcate it. 

| Vector-SIMD CFGs | Algorithm-specific Control Flow | Algorithm-specific Computation |
| :--- | :--- | :--- |
| Scalar | Contained here | Points to Vector CFG |
| Vector | Mirrors Scalar Control Flow, but with vacuous loop conditions | Contained here |

This pattern of C coding constitutes **Vector-SIMC 1.0**, in which it's the programmers responsibility to accurately separate **scalar** and **vector**. **Vector-SIMC 2.0** will include `pragma`s to support automatic **scalar**/**vector** separation.



The following example code demonstrates how the compiler uses programmer-inserted delimiters marking "loop start/end positions" in the "vector code" portion of a vector-simd program to delimit `vissue`'d blocks. These are glued in the "scalar code" portion, which contains identical delimiters in the same relative positions to mark where to glue these blocks:

```text
// program_start
// ... init code ...
while(bh1) {
  asm(while0_start)
  // ... while0 intro code block ...
    while(bh2) {
      asm(while1_start)
      // ... while1 code block ...
    }
    asm(while1_end)
  // ... while0 outro code block ...
}
asm(while0_end)
// ... outro code ...
asm(vector_return)
return;
```

The working compiler for `vvadd` currently uses an algorithm that may generalize as follows:

1. Lower the above to assembly
2. Cut the program above into blocks delimited by the nearest adjacent delimiters, removing labels and jump instructions

* example: the code between `while0_start` and `while1_start` will consist of the assembly corresponding to `// ... while0 intro code block ...` \(minus labels and jump instructions, plus junk setup code for while loops\)
* associate each block with its first delimiter \(i.e., a dictionary mapping the first delimiter to a block \[e.g. `while0_start` corresponds to `// ... while0 intro code block ...` \]\)
* program start is an implicit delimiter

3. "Glue" each blocks in the scalar code where its key \(i.e., first delimiter\) appears.

* `vector_return` is a special case: the block of code between `vector_return` and a `ret`-like instruction will be glued there.

\#TODO: need to design/implement **Vector-SIMC**

Some thoughts:

* Use Clang ASTs to parse **scalar** and **vector** from custom `pragma`s.
* Clang AST can then be lowered into either:
  * Vector SIMC version 1.0
  * MLIR 

## Gluing Scalar and Vector

After compiling the **scalar**/**vector** compilation units into **HBRISCV**, we must glue them together.

Gluing consists of the following steps:

1. Removing whitespace, comments, and "non-return" branch instructions from **vector**
   * We retain the "jump to return address" instruction, since it demarcates the return block
2. Extracting named, control-flow contiguous blocks from **vector**
   * Consecutive, control-flow contiguous blocks demarcate each other
   * Though this merges useless, control-flow setup into control-flow contiguous blocks, it's required so as to also merge any control-flow hoisted code
   * The **vector** **return block** consists of all instructions between a special `asm` op and the jump to `ra` 
3. Moving **scalar**  `vmask` ****before all other instructions to factor out function setup instructions from vector cores \(???\)
4. Pasting control-flow contiguous blocks from **vector** under the labels with the corresponding name



