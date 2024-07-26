---
title: "Python Uses Operational Chaining for Boolean Comparisons"
date: 2017-01-20T19:21:05-07:00
draft: false
comments: false
summary: It is important to understand Python's use of operational chaining to avoid programming errors.
tags:
  - python
  - computer-science
categories:
  - deep-dive
  - software
toc: true
---

## A Python feature?

A colleague noticed this behavior:

```python
>>> 'a' in 'b' == 0
False
>>> ('a' in 'b') == 0
True
>>> 'a' in ('b' == 0)
TypeError: argument of type 'bool' is not iterable
```

offering a bucket of doubloons to anyone who can explain it. The final case is simply a typecasting problem, but the first two cases demonstrate Python's operating chaining behavior.

## Explanation

### Using the Dis Module to dig deeper

Using the  `dis`[^unimportant] module, we find:

[^unimportant]: `pip install dis`


```python
'a' in 'b' == 0

  7           0 LOAD_CONST               1 ('a')
              3 LOAD_CONST               2 ('b')
              6 DUP_TOP
              7 ROT_THREE
              8 COMPARE_OP               6 (in)
             11 JUMP_IF_FALSE_OR_POP    21
             14 LOAD_CONST               3 (0)
             17 COMPARE_OP               2 (==)
             20 RETURN_VALUE
        >>   21 ROT_TWO
             22 POP_TOP
             23 RETURN_VALUE

'a' in 'b' and 'b' == 0
15           0 LOAD_CONST               1 ('a')
              3 LOAD_CONST               2 ('b')
              6 COMPARE_OP               6 (in)
              9 JUMP_IF_FALSE_OR_POP    21
             12 LOAD_CONST               2 ('b')
             15 LOAD_CONST               3 (0)
             18 COMPARE_OP               2 (==)
        >>   21 RETURN_VALUE

'a' in 'a' == 0
  9           0 LOAD_CONST               1 ('a')
              3 LOAD_CONST               1 ('a')
              6 DUP_TOP
              7 ROT_THREE
              8 COMPARE_OP               6 (in)
             11 JUMP_IF_FALSE_OR_POP    21
             14 LOAD_CONST               2 (0)
             17 COMPARE_OP               2 (==)
             20 RETURN_VALUE
        >>   21 ROT_TWO
             22 POP_TOP
             23 RETURN_VALUE
```

### In words

Parenthesis essential forces Python to do 'something' first before doing anything else. I often use parenthesis out of habit even when not needed just to be 100% clear. The above behavior shows where that habit might come in handy if Python's operation chaining isn't the desired behavior. When we put `'a' in 'b'` inside parenthesis, Python is forced to evaluate that independently of the rest of line; this evaluates to `True` or `False`, and then the `== 0` comparison acts as a Boolean comparison, giving expected behavior since Python sees False and 0 as equivalent here.

In the first instance above, however, Python is operator chaining. For `'a' in 'b' == 0`, it first loads a and b strings such that the stack is

```python
[ b, a ]
```

it then `DUP_TOP` duplicates `b` so that it doesn’t have to waste time to load it again for the second evaluation, resulting in the stack:

```python
[ b, b, a ]
```

`ROT_THREE` lifts second and third stack item up and the top item down to third place, so now the stack is:

```python
[ b, a, b ]
```

`COMPARE_OP` acts on `[ b, a]`, and since it is not true, `JUMP_IF_FALSE_OR_POP` forces it to jump and just returns False. This makes sense because it
sees this as a compound AND statement, and when the first part of an AND statement is false, the entire thing is false and it is a waste of
resources to compute further.

What is interesting is if you do `'a' in 'a' == 1`,  or `'a' in 'a' == 0`. Take the former, for example,
which after `DUP_TOP` gives us the stack:


```python
[ a, a, a ]
```

Here `'a' in 'a'` is obviously true, and these two entries of the stack are replaced with the result of that evaluation, which is `True`. This, now being the top of the stack, is popped off by `JUMP_IF_FALSE_OR_POP`
leaving the duplicated 'a' on the top of the stack, where then `1` is loaded for the next comparison, so the stack is now:

```python
[1, a]
```
and the `==` comparison is applied which is indeed False.
Thus `'a' in 'a' == 0` is chained as `'a' in 'a' and 'a' == 0` and will also be false. Similarly, `'a' in 'b' == 0` is effectively the same as `'a' in 'b' and 'b' == 0` which is false.

The following behavior, then, makes sense:

```python
>>> 'a' in 'a'
True
>>> 'a' in 'a' and 'a' == True
False
>>> 'a' in 'a' == True
False
```
