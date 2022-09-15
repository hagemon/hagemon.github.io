+++
title= "Understanding Symbolic Execution"
date= 2022-09-03T16:27:47+08:00
math= true
tags = ['Software Analyze and Testing', 'Machine Learning Testing']
draft = true
+++

The article involves personal understanding of  [a lecture about Symbolic Execution](https://www.notion.so/Understanding-Concolic-Testing-5ba0faf9cf6b4644a5eb537461f06f89), from [Jonathan Aldrich](https://www.notion.so/c9eda16fb42141389f65357c5b30a665).

---

Symbolic Execution abstract the execution of source code in a symbolic way. Unlike traditional inputs with real values, symbolic execution treats variables as symbols, which represents a set of inputs for each execution path, i.e. `if-else` in code.Symbolic execution statically parses source code line by line and generate test cases for each possible condition.

To this end, symbolic execution keeps three significant  parameters:

- $g$: track the logic of current execution path, default by true.
- $E$: store variables in current environment, note that variables are stored as symbols, and constants are stored as real values.
- stat: the code under execution. We simplify it with the line of code in this article.

For instance, assuming we have following code:

```cpp
int x=0, y=0, z=0;
if (a) {
	x -= 2;
}
if (b < 5) {
	if (!a && c) { y = 1; }
  z = 2;
}
assert(x + y + z != 3);
```

The execution of this code can be represented in a tree form or a tabular form, we use tabular form to represent one of execution path:

| stat | $g$    | $E$    |
| ---- | ---- | ---- |
| 0    | true | $a\mapsto \alpha,b\mapsto \beta,c\mapsto \gamma$    |
| 1    | true | $a\mapsto \alpha,b\mapsto \beta,c\mapsto \gamma, x\mapsto 0,y\mapsto 0, z\mapsto 0$    |
| 2    | $\neg a$    | $a\mapsto \alpha,b\mapsto \beta,c\mapsto \gamma, x\mapsto 0,y\mapsto 0, z\mapsto 0$    |
| 5    | $\neg a\wedge \beta \ge 5$    | $a\mapsto \alpha,b\mapsto \beta,c\mapsto \gamma, x\mapsto 0,y\mapsto 0, z\mapsto 0$    |
| 9    | $\neg a\wedge \beta \ge 5 \wedge 0 + 0 + 0 \ne3$    | $a\mapsto \alpha,b\mapsto \beta,c\mapsto \gamma, x\mapsto 0,y\mapsto 0, z\mapsto 0$    |

here we trigger $\neg a$, $\beta \ge 5$ in order and do not violate the assert statement.

Let’s try another path:

| stat | $g$    | $E$    |
| ---- | ---- | ---- |
| 0    | true | $a\mapsto \alpha,b\mapsto \beta,c\mapsto \gamma$    |
| 1    | true | $a\mapsto \alpha,b\mapsto \beta,c\mapsto \gamma, x\mapsto 0,y\mapsto 0, z\mapsto 0$    |
| 2    | $\neg a$    | $a\mapsto \alpha,b\mapsto \beta,c\mapsto \gamma, x\mapsto 0,y\mapsto 0, z\mapsto 0$    |
| 5    | $\neg a\wedge \beta < 5$    | $a\mapsto \alpha,b\mapsto \beta,c\mapsto \gamma, x\mapsto 0,y\mapsto 0, z\mapsto 0$    |
| 6    | $\neg a\wedge \beta < 5 \wedge \gamma$    | $a\mapsto \alpha,b\mapsto \beta,c\mapsto \gamma, x\mapsto 0,y\mapsto 1, z\mapsto 0$   |
| 7    | $\neg a\wedge \beta < 5 \wedge \gamma$    | $a\mapsto \alpha,b\mapsto \beta,c\mapsto \gamma, x\mapsto 0,y\mapsto 1, z\mapsto 2$    |
| 9    | $\neg a\wedge \beta < 5 \wedge \gamma \wedge \neg(0+1+2\ne3)$    | $a\mapsto \alpha,b\mapsto \beta,c\mapsto \gamma, x\mapsto 0,y\mapsto 1, z\mapsto 2$    |

Apparently, this path violate the assert statement and trigger a bug here. Symbolic Execution can generate a test case from $g$ to expose the problem.