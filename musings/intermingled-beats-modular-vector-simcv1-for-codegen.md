# Intermingled beats Modular Vector-Simcv1 for Codegen

Intermingled vs modular Vector-Simcv1: 

* In intermingled: the scalar steady-state prefetch loop is mixed with the vector while loop and vector blocks via \#defines.
  * Pros:
    * intermingled is very simple target for splitter:
      * supports incremental development
    * Cons:

      * somewhat awkward coding convention: vector code lives in steady-state prefetch loop
      * somewhat less readable
* In modular: scalar steady-state loop structure and vector blocks are copied by hand into a "vector module" 
  * Pros: 
    * Somewhat less awkward coding convention
    * Somewhat more readable output
  * Cons: 
    * modular is difficult target for splitter:
      * need to separate steady-state control-flow \(impossible with FSM\) from vissue blocks
      * requires vissue block / scalar loop pragmas, which is many steps ahead of where we are
      * demands all-at-once development

