---
description: >-
  To perform gluing, lotus makes a lot of assumption; here we collect many of
  them. These assumptions arise from working with a vvadd kernel, they may not
  generalize beyond that.
---

# Tricking the Compiler: \`gcc\` assumptions

Observations / Tricks / Shortcuts:

* **`ret` after last label**: We use C-level labels in **scalar** to mark labels from which to vissue. These labels are placed after the `return` , so that we never actually execute them. However, a `ret` is still emitted after the last label; we'll assume it's always emitted \(and not a `jr ra` , for e.g.\) and simply remove it.
* **control-contiguous blocks contain junk control logic**: compiler-hoist code must be included in control-contiguous blocks that preface or postface a loop. In a preface block we take a conservative approach by capturing all code from the start of the containing control body up to the beginning of the inner control body. For instance, the code between `while0_start` and `while1_start` corresponds to the preface \(intro\) block for the outer while loop in the following code:

  ```text
  // program_start
  // ... init code ...
  while(bh1) {
    asm(while0_start)
    // ... while0 intro code block ...
      while(bh2) {
        asm(while1_start)
        // ... while1 code block ...
        asm(while1_end)
      }
    // ... while0 outro code block ...
    asm(while0_end)
  }
  asm(vector_return)
  return;
  ```

  Notice that this block will contain setup code for the contained `while`-loop header; this won't affect correctness so long as we ignore jump statements. However, this junk control logic will affect performance. Similar reasoning is applied to postface \(outro\) blocks.

