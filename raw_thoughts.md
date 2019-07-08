  + [Avoid Bitwise Operations for Booleans](#avoid-bitwise-operations-for-booleans)

### Avoid Bitwise Operations for Booleans

Sometimes I see code like this
```c++
bool b = f();  // Function `f`  returs `bool`.
..             // Some code here.
b &= f2();     // Function `f2` returs `bool`.
```
The author wants something like this `bool b = f() && f2()`, but some code needs to be executed between `f()` and `f2()`. The line with the operator `&=` is [equivalent](https://en.cppreference.com/w/cpp/language/operator_assignment#Builtin_compound_assignment) to the line `b = b & f2()`. Here the operator `&` is _bitwise_ and in general case is different from the _logical_ operator `&&`.

As long as in the fragments above the `b` is correct `bool`, `f()` and `f2()` return correct `bool`, the behavior is likely to correspond to the author's expectation. But some programmers use the line with the operator `&=` as a model, and they apply it for functions returning _bool-like_ value, i.e. for functions that return `0` as an indication of `false`, and an _arbitrary non-zero value_ as an indication of `true`.  
Now imagine how the behavior of the line can change:
```c++
// Let's assume that by this moment `b` is `true` (1).
b &= f3();  // Function `f3()` returns `4` as an indication of `true`.
```
The line is equivalent to `b = b & f3()`, which is equivalent to `b = 1 & 4`, which results in a value of `0` (`false`) saved in `b`. To summarize, the author is trying to apply the AND operation to `true` and true-like value `4`, expects `true` as a result, but gets `false`. The compiler will hardly warn.

To avoid problems use the logical operators instead, e.g. `b = b && f2()`. The logical operators (`&&`) require `bool` operands (`b` and value returned by `f2()`). If any operand is not `bool` then it will be converted to `bool` according to the [Boolean Conversion rules](https://en.cppreference.com/w/cpp/language/implicit_conversion#Boolean_conversions). And the result will correspond to the expectation.

__Note__  
However the recommended logical operators still may not fully filter out the _incorrect `bool`_ values (not equal to `false` and not equal to `true`) shown in the section [Know the Limitations of `memset()` When Initializing](#know-the-limitations-of-memset-when-initializing). E.g. 
```c++
b = b || f4();  // Function `f4()` returns an incorrect `bool` with value `0x0101`.
```
The compiler can still implement the logical operation `||` with the bitwise OR machine instruction applied to `b` and value returned by `f4()`, and save the result in `b`, leading to an incorrect value `0x0101` in `b`.

__What To Remember__
* Avoid bitwise operators for Booleans. Use logical operators instead. E.g. `b = b && f2()`.
* Don't forget that the right-hand-side of the operator can be optimized out. E.g. if `b` is `false` then the result of `b && f2()` will also be `false` regardless of `f2()`. That is why the call to `f2()` can be skipped in some contexts (and in some contexts it _will_ be skipped [to do]). See details in [[MISRACpp2008]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#MISRACpp2008) Rule 5–14–1 mentioned below.

__How to Automate Catching This__  
The code analysis tools supporting the following checks should catch this issue.  
* [[MISRACpp2008]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#MISRACpp2008) _MISRA C++:2008 - Guidelines for the use of the C++ language in critical systems_.
  * Rule 4–5–1 (Required) Expressions with type _bool_ shall not be used as operands to built-in operators other than the assignment operator =, the logical operators &&, ||, !, the equality operators == and !=, the unary & operator, and the conditional operator.
* [[MISRAC2004]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#MISRAC2004) _MISRA-C:2004. Guidelines for the use of the C language in critical systems_.
  * Rule 12.6 (advisory): The operands of logical operators (&&, || and !) should be effectively Boolean. Expressions that are effectively Boolean should not be used as operands to operators other than (&&, ||, !, =, ==, != and ?:).

__See Also__
* [V564](https://www.viva64.com/en/w/v564/). The '&' or '|' operator is applied to bool type value. You've probably forgotten to include parentheses or intended to use the '&&' or '||' operator (+[RU](https://www.viva64.com/ru/w/v564/)).
* [V792](https://www.viva64.com/en/w/v792/). The function located to the right of the '|' and '&' operators will be called regardless of the value of the left operand. Consider using '||' and '&&' instead (+[RU](https://www.viva64.com/ru/w/v792/)).
* [[MISRACpp2008]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#MISRACpp2008) _MISRA C++:2008 - Guidelines for the use of the C++ language in critical systems_.
  * Rule 5–0–20 (Required) Non-constant operands to a binary bitwise operator shall have the same _underlying type_.
  * Rule 5–0–21 (Required) Bitwise operators shall only be applied to operands of unsigned _underlying type_.
  * Rule 5–14–1 (Required) The right hand operand of a logical `&&` or `||` operator shall not contain side effects.
