---
description: >-
  An unresolved difficulty that emerges if the scalar code decides it wants to
  do some work.
---

# Scalar Code + Hoisting = Doomed Orphans  \[EDIT: THIS IS A NON-PROBLEM\]

EDIT: The situation described here is not actually a problem, since SCALAR CODE DOES NOT EXIST IN **vector** \(facepalm\).

Two features conspire to create a dilemma:

* Loop-invariant code motion \(i.e., hoisting\)
* arbitrary scalar code

Consider a modular delimiting scheme using start and end delimiters to mark `vissue`  blocks: 

```c
asm(vector_init_start) 
/* vector_init code */ 
asm(vector_init_end) 
while(bh) { 
    asm(vector_body_start) 
    /* vector_body code */ 
    asm(vector_body_end) 
} 
asm(vector_return) 
return;
```

Hoisting introduces a key problem in the emitted assembly:  `vector_body` code may be hoisted out of the while loop body and become "orphaned", living neither in the `vector_init` nor in the `vector_body` blocks. We circumvent this key problem and **save the orphans** by dumping all of the assembly emitted in the control-flow boundary into the `vector_init` block, filtering out jumps, and probably keeping useless control-flow setup code. Oh well, at least the orphans are safe.  
  
However, if we instead had the code below, this strategy breaks down \(this isn't the structure of vvadd, but bear with me\):  

```c
asm(vector_init_start) 
/* vector_init code */ 
asm(vector_init_end)
f();
while(bh) { 
    asm(vector_body_start) 
    /* vector_body code */ 
    asm(vector_body_end) 
} 
asm(vector_return) 
return;
```

No longer can we save the orphans by dumping all the code in the control-flow boundary into the outer block, since this would also include the scalar call to f. 

Delimiters that mark scalar code might fix this problem:

```c
asm(vector init_start) 
/* vector_init code */ 
asm(vector init_end)
asm(scalar f_start)
f();
asm(scalar f_end)
while(bh) {
    asm(vector body_start) 
    /* vector_body code */ 
    asm(vector body_end) 
} 
asm(vector_return) 
return;
```

Now we have two kinds of delimiters: vector and scalar, and we can simply place hoisted code in the nearest, outermost vector block.

Here's the kicker: if we allow scalar code in loop bodies, I don't know how to save the orphans!!! 

```c
asm(vector init_start) 
x_vec = ...
asm(vector init_end)
asm(scalar init_start)
x_sc = ...
asm(scalar init_end)
while(bh) {
    asm(scalar body_start)
    int y_sc = 2*x_sc
    asm(scalar body_end)
    asm(vector body_start) 
    int y_vec = 2*x_vec 
    asm(vector body_end) 
} 
asm(vector_return) 
return;
```

In the context of arbitrary scalar code, each hoisted "orphaned" instruction belongs to a particular type of home: it belongs to either a scalar parent or a vector parent, and it's unclear to me how to identify them as such. For example, in the above, if the initialization of `y_sc` is hoisted, the orphaned instructions must be relocated into a scalar block; they are **scalar orphans**. Correspondingly, `y_vec` are **vector orphans** belonging to a vector block.

At this point, I don't have any good ideas on how to differentiate the two. The compiler that assigns all non-control-flow code to vector cores avoids this issue since, we know that all orphaned instructions are vector orphans belong to the nearest, outermost \(vector\) block. We'd like to solve this problem so as to implement first-order `vissue` ing a'la [Adrian's post](http://adriansampson.net/notes/bhdk6k02ztv0/#an-imaginary-programming-model).

