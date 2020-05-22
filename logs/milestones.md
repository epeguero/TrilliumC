---
description: >-
  Milestones and action steps towards having a useful language for implementing
  benchmarks.
---

# Trillium Implementation Milestones

{% file src="../.gitbook/assets/scribble.pdf" caption="Path to \`offload\` Semantics" %}

After each step below, test that it works and perform a pull-request \(delete unnecessary files beforehand, so that PR only shows relevant changes\):

**Splitter Pass 1**:

* [x] Correct intermingled `vvadd_kernel.c` for compatibility with `glue.py` 
  * `modular_kernel.c` is available in case things go crazy: you can gradually transform it into the intermingled version, and see what complications arise
  * merging scalar end boilerplate fully into scalar pragma:
  * adding scalar return marker
  * moving fence and return
  * moving vissue labels
  * adding vector return marker after scalar boilerplate
* [ ] kernel start \(i.e., _**\#pragma trillium vec\_simd begin vvadd**_\)
  * no-op; require function signature 
* [ ] kernel end \(i.e., _**\#pragma trillium vec\_simd end vvadd**_\)
  * scalar \(devec, fence, return marker, return, vissue labels\)
  * vector \(return marker, return\)
  * don't emit end curly brace

**Gluer Pass 1**:

* [ ] Give names to vector block delimiters:
  * `until_next`
  * `return` 
  * actually parse vissue block keys, rather than looking for hardcoded "vector init", "vector body", ..
  * mark each vissue key with prefix "trillium vissue"
  * as before, glue the vissue block \(in vector.s\) where the corresponding vissue key lies \(in scalar.s\)
  * IMPORTANT QUESTION: how are vissue blocks delineated in general?
* [ ] Add `begin/end` block delimiters:
  * represent the obvious, "naive" approach to delimiting vissue blocks
* **Splitter Pass 2**:
* [ ] Remove the need for labels altogether
  * Vector-SIMCv2 enhancement: replace vissue instructions with psuedo-vissue instructions, with single param being the vissue key itself \(rather than a label pointing to a vissue key\)
  * Generate a label for each vissue key, and place that key under it
  * at this point, most boilerplate has been removed from Vector-SIMCv2
* [ ] Remove the need for vissue keys altogether
  * _**pragma vector begin/end marks a vissue block**_
  * scalar boilerplate generation: 
  * generate a vissue key and label for it 
  * call vissue on the label
  * place key under label
  * vector boilerplate generation:
  * _**pragma trillium scalar loop: reflect for loop as while loop**_
  * generate blackhole condition for while loop
  * delineate code block using vissue key
  * place generated vissue key under generated label

At this point, we have a serviceable language for building benchmarks. Afterwards, we can focus on building benchmarks and/or generating prefetching.

* [ ] consider porting `glue.py` to Rust or Haskell at this point
  * Opportunity to generalize machinery linewise FSM-parser from `splitter.hs` 
  * Opportunity to learn about Rust and get a feel for "cost-free abstractions"

