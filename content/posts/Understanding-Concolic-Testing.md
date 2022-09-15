+++
title= "Understanding Concolic Testing"
date= 2022-09-03T16:10:21+08:00
math= true
tags = ['Software Analyze and Testing', 'Machine Learning Testing']
draft = true
+++

The article involves personal understanding of  [a lecture about Concolic Testing](https://www.cs.cmu.edu/~aldrich/courses/17-355-19sp/notes/notes15-concolic-testing.pdf), from [Jonathan Aldrich](https://www.notion.so/Understanding-Concolic-Testing-5ba0faf9cf6b4644a5eb537461f06f89).

---

There exists two mainly challenges in software testing:

1. designing a set of test cases that cover all the source code.
2. finding inputs that trigger corner case defects.

Symbolic execution is a classic method to solve problems above, but it suffers from some significant limitations:

- SMT solvers may not be able to find satisfying assignments to variables for long paths with many conditions.
- SMT solvers can not cover paths that require high computational effort.

Concolic testing overcomes these problems by combining **con**crete execution and symb**olic** execution.  We present a clear example here to show how concolic testing works.

```cpp
int foo(int v) {
    return 2*v;
}

int 

void bar(int x, int y) {
    z = foo(y);
    if (z == x) {
        if (x > y+10) {
            ERROR;
        }
    }
}
```

To test function `bar`, concolic testing (shorten as CT) starts with random inputs and execution the program to the corresponding path, e.g. $x=22,y=7,z=14$ and $z \ne x$. CT then symbolically execute this path and come up with $g:2*y_0\ne x_0$. After that, CT asks SMT to produce a input that violate $g$ to access another path, until all the paths are discovered. Apparently, errors would be discovered during this process.

Concrete steps are described as:

1. random inputs $x=22, y=7$;
2. calculate $z = 14$;
3. symbolic execution on this path:
    1. $x=x_0,y=y_0$;
    2. $z=2*y_0$;
    3. path condition $g:2*y_0 \ne x_0$;
4. explore other paths against executed path conditions in 2:
    1. negate the path condition and get $g:2*y_0==x_0$;
    2. ask SMT to produce a corresponding solution, e.g. $x_0=2,y_0=1$;
    3. replace input with new solution and run 1-3.

A more complex example with a high computational effort function `foo` is presented as following code:

```cpp
int foo(int v) {
    return v*v%50;  // this may lead high computational effort.
}

void baz(int x, int y) {
    z = foo(y);
    if (z == x) {
        if (x > y+10) {
            ERROR;
        }
    }
}
```

The statement  `v*v%50`is a not linear and may defeat the SMT solver.

To solve this problem, CT replace the condition path $x_0\ne(y_0*y_0)\%50$ in traditional symbolic execution with concrete inputs value.

Concretely in the symbolic state $z=foo(y_0)$，CT use $y_0=7$ for the following state $foo(y_0)==x_0$ and direct execute $x_0==49$ and find error inside the block.