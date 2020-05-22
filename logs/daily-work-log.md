# Daily Work Log

## May 7

* Meeting with Neil and Phil
  * discussed the [potential limitations](https://app.gitbook.com/@edwinmpeguero/s/hbir/~/drafts/-M6mXObEDE_1tyszqgCp/musings/the-problem-with-risc-like-targets-for-parallel-architectures) of embedding a Vector-SIMC model in C
  * got `vvadd` output of `glue.py` working by adding one missing instruction:
    * An initial stack manipulation was missing from the first vector block, since it occurred before the delimiting label `vector_init`
* Implemented this last piece of the puzzle into  `glue.py` .
* Began sketching a finite state machine parser for a more general version of `glue.py` that uses only delimiters placed at control-flow boundaries \(e.g., at beginning or end of a while loop\)

## May 8

* [Documented](https://app.gitbook.com/@edwinmpeguero/s/hbir/~/drafts/-M6pyX1wartJYl3bK1x_/musings/scalar-code-+-hoisting-doomed-orphans) thoughts on a potential issue that arises when scalar code does algorithmic work.
  * This issue is actually an obvious non-issue, since scalar code will not be written in **vector**.
* Met with Adrian, discussed next steps:
  * Compiler consists of a **splitter** and a **gluer**.
  * **splitter** takes a high-level \(Vector-SIMC v2.0\) Vector-SIMD specification, and emits **vector** and **scalar** code \(Vector-SIMC v1.0\).
  * **gluer** takes **vector** and **scalar** \(Vector-SIMC v1.0\) and emits a low-level Vector-SIMD program \(HBRV\).

