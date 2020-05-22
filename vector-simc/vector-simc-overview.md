---
description: A high-level overview of programming Vector-SIMD in Trillium.
---

# Overview: Vector-SIMD in Trillium

We decompose the **trillium** compiler into two subsystems, **gluer** and **splitter**:

**trillium**\(source : _**Vector-SIMC v2.0**_\) = **gluer**\(**splitter**\(source\)\)

* **gluer:** _**Vector-SIMC v1.0**_ **--&gt;** _**HBRV**_
  * lowers _**Vector-SIMC v1.0**_ into _**HBRV**_ vector/scalar code and combines these into single Vector-SIMD _**HBRV**_ program
* **splitter:** _**Vector-SIMC v2.0**_ **--&gt;** _**Vector-SIMC v1.0**_ \(intermingled\)
  * lowers _**Vector-SIMC v2.0**_ to intermingled version of _**Vector-SIMC v1.0**_

_**Vector-SIMC v1.0**_: preprocessor directive+inline assembly style programming for Vector-SIMD

```c
// KERNEL NAME:
// kernel signature always returns void and passes int `mask`
void my_kernel(int mask) {
  // BOILERPLATE:
  // preprocessing directives marking scalar/vector code
  #ifdef SCALAR
  
  // BOILERPLATE: 
  // the start of scalar code requires a call to VECTOR_EPOCH, 
  // in order to disable vector core I-caches,  
  // thereby turning them into actual vector cores
  // NOTE: See Phil for info on how to construct `mask`
  VECTOR_EPOCH(mask);
  
  // KERNEL: 
  // consists of arbitary scalar code, `vissue`s, and prefetching.
  // At this stage in compiler development, we don't handle prefetching
  for(int i = f(x)...) {
    // example vissue, calling vector code block `do_stuff` 
    // across vector cores
    ISSUE_VINST(do_stuff)
  }
  
  // BOILERPLATE: 
  // returning from scalar code
  // requires these four, consecutive lines
  // 1. scalar core reactivates vector cores' I-caches 
  DEVEC(devec_0); 
  // 2. all cores wait for each other
  asm volatile("fence");
  // 3. marker for scalar stack cleanup assembly, 
  // to be moved to before DEVEC by gluer
  asm("scalar return"); 
  // 4. C function return
  return;
  
  // VISSUE KEYS:
  // labels pointing to vector blocks
  do_stuff:
    asm("do_stuff");
  // as many labels as you'd like... 
  
  
  // BOILERPLATE: see `#define` above
  #elif defined VECTOR
  
  // VARIABLE INITIALIZATION:
  // initialization of variables used across all code blocks.
  // Also includes "blackhole" (named `bh`) reflected loop conditions,
  // which must be volatile and uninitialized.
  volatile int bh;
  /* decls of vars used below */ 
  
  // REFLECTED KERNEL: 
  // reflection of scalar's control-flow structure.
  // scalar `for` and `while` loops become vector `while` loops, 
  //   with "blackhole" loop conditions (e.g., `bh`).
  // the code for every vissue actually appears in this reflection.
  while(bh) {
    // VECTOR BASIC BLOCK:
    // corresponds to `ISSUE_VINST(do_stuff)` in this reflected kernel.
    asm("do_stuff start")
    /* code block for do_stuff */
    /* NOTE: cannot contain control_flow */
    asm("do_stuff end")
  }
  
  // BOILERPLATE:
  // returning from vector code requires these two lines:
  // 1. marker for vector stack cleanup assembly
  asm("vector return");
  // 2. C function return
  return;
}
```

* User-defined Vector-SIMD kernel parameters:
  * kernel name \(line 3\)
  * **scalar:**
    * kernel \(between lines 15-22\)
    * VISSUE keys \(lines 37-41\)
  * **vector**:
    * variable initialization \(lines 51-52\)
    * reflected kernel \(lines 54-66\)
      * reflected control-flow \(lines 59-66\)
      * vector code blocks for each **scalar** `vissue` \(lines 62-65\) 
* Boilerplate:
  * kernel function signature
  * `#ifdef SCALAR ... #elif defined VECTOR` preprocessing directives
  * scalar and vector returns 
* Static constraints:
  * **vector** reflected kernel must actually be reflection of **scalar** kernel
    * `for` loops must become "blackhole" `while` loops
    * `vissue`s must become the code being `vissue`d
      * no other code allowed in reflected kernel
  * any variables mentioned in **vector** reflected kernel must be initialized in **vector** variable initialization 

This boilerplate simplifies manually coding in _**Vector-SIMC v1.0.**_   
Equivalently, _**intermingled Vector-SIMC v1.0**_ merges the kernel and its reflection by intermingling preprocessor directives:

```c
// KERNEL NAME:
// kernel signature always returns void and passes int `mask`
void my_kernel(int mask) {
  #ifdef SCALAR
  
  // BOILERPLATE: 
  // the start of scalar code requires a call to VECTOR_EPOCH, 
  // in order to disable vector core I-caches,  
  // thereby turning them into actual vector cores
  // NOTE: See Phil for info on how to construct `mask`
  VECTOR_EPOCH(mask);
  
  #elif defined VECTOR
  // VARIABLE INITIALIZATION:
  // initialization of variables used across all code blocks.
  // Also includes "blackhole" (named `bh`) reflected loop conditions,
  // which must be volatile and uninitialized.
  volatile int bh;
  /* decls of vars used below */
  #endif
  
  /* decls of vars used below */ 
  // KERNEL: 
  // consists of arbitary scalar code, `vissue`s, and prefetching.
  // At this stage in compiler development, we don't handle prefetching
  #ifdef SCALAR
  for(int i = f(x)...)
  #elif defined VECTOR
  while(bh)
  #endif
  {
    #ifdef SCALAR
    // example vissue, calling vector code block `do_stuff` 
    // across vector cores
    ISSUE_VINST(do_stuff)
    
    #elif defined VECTOR
    // VECTOR BASIC BLOCK:
    // corresponds to `ISSUE_VINST(do_stuff)` in this reflected kernel.
    asm("do_stuff start")
    /* code block for do_stuff */
    /* NOTE: cannot contain control_flow */
    asm("do_stuff end")
    #endif
  }
  
  #ifdef SCALAR
  // BOILERPLATE: 
  // returning from scalar code
  // requires these four, consecutive lines
  // 1. scalar core reactivates vector cores' I-caches 
  DEVEC(devec_0); 
  // 2. all cores wait for each other
  asm volatile("fence");
  // 3. marker for scalar stack cleanup assembly, 
  // to be moved to before DEVEC by gluer
  asm("scalar return"); 
  // 4. C function return
  return;
  
  // VISSUE KEYS:
  // labels pointing to vector blocks
  do_stuff:
    asm("do_stuff");
  // as many labels as you'd like... 
  
  #elif defined VECTOR
  // BOILERPLATE:
  // returning from vector code requires these two lines:
  // 1. marker for vector stack cleanup assembly
  asm("vector return");
  // 2. C function return
  return;
}
```

**splitter** targets this intermingled version, since it's simpler to emit: 

* the non-intermingled version requires first parsing the entire scalar kernel control-flow structure, reflecting it, and then placing these two pieces where they belong. 
* the intermingled version allows reflecting on-the-fly by simply swapping `for` loops to `while` loops.

_\*\*\*\*_

_**Vector-SIMC v2.0**_: C custom-pragma style programming for Vector-SIMD   
  
The following Vector-SIMC v2.0 kernel is equivalent to the above Vector-SIMC v1.0 kernel: 

```c
// demarcates the start/end of a vector-simd kernel
#pragma trillium vec_simd my_kernel 16 start

  // KERNEL: 
  // consists of arbitary scalar code, `vissue`s, and prefetching.
  // At this stage in compiler development, we don't handle prefetching
  for(int i = ...f(x)...) {
    // demarcates the start/end of a vector basic block
    #pragma trillium vector begin
    /* code block for do_stuff */
    /* NOTE: cannot contain control_flow */
    #pragma trillium vector end
  }

#pragma trillium vec_simd end
```

In _**Vector-SIMC v2.0**_, we remove the need for both the boilerplate and static constraints via custom pragmas.  


User-defined Vector-SIMD kernel parameters:

* kernel name
* kernel
  * vector blocks
  * arbitrary scalar code

To simplify boilerplate generation, we reserve the following keywords:

* `mask` 
* `devec_0`
* `bh_n` , for all n &gt;= 0
* `vissue_key_n`, for all n &gt;= 0

The presence of these inside the kernel will produce undefined behavior.

