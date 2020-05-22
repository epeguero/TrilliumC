# Minutes Of Meeting



## May  14: Phil, Neil, Edwin

* Agreed that adding a "passed" button png to github is key to being legit
* Phil changed how to build groups:
  * `vector_group_template_n` builds \(n/2\)x\(n/2\) sized vector group in grids with size that's a multiple of n
    * 4 and 16 versions implemented
* Phil mentioned we should start using 64 cores when running benchmarks
* Phil added config bit to prefetch: 0 for horizontal load, 1 for vertical
* Edwin asked Phil how we reduce the number of PREFETCHES
  * Phil showed a graph demonstrating how runtime decreases with Prefetch width
* Phil walked us through a simplification to vector core memory management 
  * Before: each load would check if their frame was ready, and remem clear entire frame
  * Now: the waiting is factored out, and vector cores can remem bits at a time
    * FRAME\_START\(REGION\_SIZE\) wait for REGION\_SIZE amount of frame to be ready
    * REMEM\(REGION\_SIZE\) clear REGION\_SIZE amount of data
* Neil asked in which branch to find these changes:
  * Main changes are in scalar
* Edwin asked what action steps to take to get onto the scalar branch
  * Neil recommended: 

      1\) pull and merge scalar into glue: 

    ```text
      git checkout scalar
      git pull
      git checkout glue 
      git pull glue
      git merge scalar
      git push glue
    ```

      2\) issue pull request to Phil

  * Neil will merge onto scalar, after which Edwin can merge without conflict \(hopefully\)

## May 15: Adrian, Edwin

* Edwin discussed new internship opportunities with Adrian
* Edwin confessed that he's moved from Python back into Haskell world
  * Adrian recommended that I check out Rust
    * It has "traits", which are like typeclasses!
    * It's functional!
    * It's super fast \(low-cost abstractions!\)
* Adrian recommended that Edwin push trillium version 1.0 to repo once the following features work:
  * vvadd runs through splitter/gluer:
    * while loop generation
    * boilerplate
  * runs through gem5, emits `[[SUCCESS]]`
    * `make` test
* Adrian found that there's a tornado warning

## May 19: Adrian, Neil, Phil

* Edwin missed the meeting, due to a rescheduling
* Discussed a deadline coming up in two months, and the importance of getting good benchmarks going
* As a corollary: the importance of getting a good language to write benchmarks

## May 21: Edwin, Neil, Phil

* Phil discussed implementing manycore reduction
  * he mentioned it's not worth implementing the thing where you iteratively reduce by a factor of 2 
* Work on the splitter until end of the week; focus on glue next week
* Phil made a git wiki for `gem5-mesh` with info on how to program on gem5
  * We can add pages for how to program in Vector-SIMCv1/2
* Make a note of optimizations to fix later; focus on getting working compiler
* Edwin merged `glue` branch with `scalar`

