+++
title= "Understanding Symbolic Execution"
date= 2022-09-03T16:27:47+08:00
math= true
+++

# Understanding Symbolic Execution

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

| stat | ⁍    | ⁍    |
| ---- | ---- | ---- |
| 0    | true | ⁍    |
| 1    | true | ⁍    |
| 2    | ⁍    | ⁍    |
| 5    | ⁍    | ⁍    |
| 9    | ⁍    | ⁍    |

here we trigger $\neg a$, $\beta \ge 5$ in order and do not violate the assert statement.

Let’s try another path:

| stat | ⁍    | ⁍    |
| ---- | ---- | ---- |
| 0    | true | ⁍    |
| 1    | true | ⁍    |
| 2    | ⁍    | ⁍    |
| 5    | ⁍    | ⁍    |
| 6    | ⁍    | ⁍    |
| 7    | ⁍    | ⁍    |
| 9    | ⁍    | ⁍    |

Apparently, this path violate the assert statement and trigger a bug here. Symbolic Execution can generate a test case from $g$ to expose the problem.