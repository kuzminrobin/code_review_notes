Know the Danger of Overflowing (and Underflowing) the Signed Types
-
The _signed_ and _unsigned_ integer types behave differently when overflown or underflown.  

First let's consider by the example of the _unsigned_ types.  
If we have an _unsigned_ 8-bit integer (whose range is from 0 (0x00) to 255 (0xFF)) with value 0, and we decrement it (thus causing the underflow), then we get the value 255 (0xFF). Fully defined value and fully defined behavior.
In the same way incrementing the value 255 (0xFF) (thus causing the overflow) will result in the value 0 (0x00). Also a fully defined value and fully defined behavior.  

C++98:  
> __3.9.1 Fundamental types__  
> 4: Unsigned integers, declared unsigned, shall obey the laws of arithmetic modulo 2^n where n is the number of bits in the value representation of that particular size of integer. Footnote 41)  
> 41) This implies that unsigned arithmetic does not overflow because a result that cannot be represented by the resulting unsigned integer type is reduced modulo the number that is one greater than the largest value that can be represented by the resulting unsigned integer type.

But for the _signed_ integers the picture is completely different.  
If we have a signed 8-bit integer, e.g. `int8_t` (with the range  
from `std::numeric_limits<int8_t>::min()` in C++, or `INT8_MIN` in C,  
to `std::numeric_limits<int8_t>::max()` in C++, or `INT8_MAX` in C),  
with the lowest value `std::numeric_limits<int8_t>::min()` (`INT8_MIN`) and we decrement such a value then we get the _Undefined Behavior_!  
The same is for incrementing the highest value `std::numeric_limits<int8_t>::max()` (`INT8_MAX`).  
It is emphasized, not just the _value_ is undefined, but the whole _behavior_ is undefined.  

The same is for the floating point types.  

C++98:  
> __5 Expressions__  
5: If during the evaluation of an expression, the result is not mathematically defined or not in the range of representable values for its type, the behavior is undefined, unless such an expression is a constant expression (5.19), in which case the program is ill-formed.

__This all means the following:__  
All the calculations that rely on the integer type overflowing, e.g. the checksums (CRC), hash codes, ring buffer indices, must use the _unsigned_ types.  
Negating the signed value (`-signedValue`) and  
  + assigning such a value to oneself (`x = -x;`) or to a variable of the same type (`y = -x;`)
  + or passing such a value as an argument (corresponding to the parameter of the same type), e.g.  
    `void foo(int y); main() { int x =..; foo(-x); }`  

can cause overflow, i.e. Undefined Behavior, if before the negation the value was `std::numeric_limits<int??_t>::min()` (`INT??_MIN`). This is because negating such a value can result in a value (`std::numeric_limits<int??_t>::max()` + 1) (`INT??_MIN` + 1) which does not fit in the type.  

One of the reasons why unsigend and signed integers behave so differently is the fact that the signed integers have at least [3 implementation-dependent representations](https://softwareengineering.stackexchange.com/questions/239036/how-are-negative-signed-values-stored/239039#239039) in C. [This paper](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2218.htm) tells that there is an intention to use in C and C++ only one representation for the signed integers (which allows using the same algorithms and/or hardware logic both for signed and unsigned integer arithmetic (at least addition and subtraction, and very likely multiplication and division)). After that intention ends up in the C and C++ Standards the overflow and underflow of the signed integers are expected to become fully defined (the same way as for the unsigned integers).  

__More Info:__  
* C++Now 2018 Closing Keynote: Undefined Behavior and Compiler Optimizations, _John Regehr_ ([abstract](http://sched.co/ELaF), [slides](https://github.com/boostcon/cppnow_presentations_2018/blob/master/05-11-2018_friday/undefined_behavior_and_compiler_optimizations__john_regehr__cppnow__05112018.pdf), [video](https://youtu.be/AeEwxtEOgH0)).  
* C++Now 2014 Undefined Behavior in C++: What is it, and why do you care? _Marshall Clow_ ([video](https://www.youtube.com/watch?v=uHCLkb1vKaY)).  

Info Sources About the Exceptions
-
Partially based on Deb Haldar's _Top 15 C++ Exception handling mistakes and how to avoid them_ [[t15c++ehm]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md), _Where do we go from here?_ section.

__Books and Articles at a High Level__
* `(Ongoing )` [C++ Exception FAQ](https://isocpp.org/wiki/faq/exceptions) on isocpp.org (still TODO).
* `1996.??.??` [More Effective C++ – 35 new ways to improve your programs and designs](https://www.amazon.com/More-Effective-Improve-Programs-Designs/dp/020163371X/ref=sr_1_1?ie=UTF8&qid=1470238686&sr=8-1&keywords=more+effective+c+meyers) – items 9 through 15 [[MEC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md).
* `1999.11.18` [Exceptional C++ – 47 Engineering puzzles, programming problems and solutions](https://www.amazon.com/Exceptional-Engineering-Programming-Problems-Solutions/dp/0201615622/ref=sr_1_1?ie=UTF8&qid=1470238861&sr=8-1&keywords=Exceptional+C%2B%2B) – items 8 through 19 [[ExcC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md).
* `2004.10.??` [C++ Coding Standards – 101 Rules, Guidelines and Best Practices](https://www.amazon.com/Coding-Standards-Rules-Guidelines-Practices/dp/0321113586/ref=sr_1_1?ie=UTF8&qid=1470238746&sr=8-1&keywords=C%2B%2B+Coding+standards) – items 68 through 75 [[C++CS]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md).
* `2015.??.??` [[e&su]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md) MSDN. _Exceptions and Stack Unwinding in C++_.
* `2016.08.03` [[t15c++ehm]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md) Deb Haldar. Top 15 C++ Exception handling mistakes and how to avoid them.

__Videos__
* `2018.05.09` C++Now 2018: Michael Spencer, ["How Compilers Reason About Exceptions"](https://cppnow2018.sched.com/event/EC7V/how-compilers-reason-about-exceptions?iframe=no&w=100%&sidebar=yes&bg=no) (to go online in 2018.06).
* `2018.05.09` C++Now 2018: Phil Nash, ["Option(al) Is Not a Failure"](https://cppnow2018.sched.com/event/EC7P/optional-is-not-a-failure) (to go online in 2018.06). Presentation [links](http://levelofindirection.com/refs/cpp-optional.html).

__Internals and ABIs__
* `2011.01.10` [[.eh_f]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md) Airs – Ian Lance Taylor. _".eh_frame"_.

Certain Code Fragments Should Not Throw Exceptions
-
In general that's the code that is executed from the moment when the execption is thrown and until entering the `catch`-handler (i.e. during the _stack unwinding process_, see [[e&su]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md)). Some of that code is implicitly generated by the compiler (hiddenly from the programmer).  
The programmer-written code fragments are  
* _destructors_, see  
Meyers' [[EC++3]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md), _Item 8: Prevent exceptions from leaving destructors_,  
Sutter's [[ExcC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md), _Item8: Writing Exception-Safe Code - Part1_.  
* `operator delete`, `operator delete[]`, see  
Haldar's [[t15c++ehm]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md), _Mistake # 5: Throwing exceptions in destructors or in overloaded delete or delete[] operator_.  
* _constructors of the exception classes_, see  
Haldar's [[t15c++ehm]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md), _Mistake # 12: Throwing exception in an exception class constructor_.

C++ Exception Specifications
-

The Herb Sutter's article ["A Pragmatic Look at Exception Specifications"](http://www.gotw.ca/publications/mill22.htm) ([[pl@es]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md)) makes think that the exception specifications  
* instead of providing the _compile-time_ nothrow-guarantee (`throw()`) or throws-only-these-ones-gaurantee (`throw(A, B)`)  
* provide the _run-time_ checks/asserts (which are _pessimization_, i.e. they lower down the performance).  

Another confirmation is in [[eses]](https://github.com/kuzminrobin/code_review_notes/edit/master/article_list.md).  

The Meyers' [[EMC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md), _Item 14: Declare functions `noexcept` if they won’t emit exceptions_,  
in the beginning makes one think that `noexcept` helps to optimize in certain cases (by means of replacing the copy operations with the move operations). But in the end still shows that there are no any _copile-time_ guarantees (a `noexcept`-function can call the functions without the exception specification, and the compilers (so far) don't complain) but there are _run-time_ checks ("_your program will be terminated if an exception tries to leave the function_").  
See also [t15c++ehm], [_Mistake # 9: Not realizing the implications of "noexcept" specification_](http://www.acodersjourney.com/2016/08/top-15-c-exception-handling-mistakes-avoid/).  

----
See also:  
* [[t15c++ehm]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md); [[MEC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md), _Item 14: Use exception specifications judiciously_.

C++ Exceptions: TODO.
-
* Concise high-level explanation of what the C++ exceptions are for, is it possible to live without the exceptions (with the exceptions completely disabled at compile time)? If yes then what is the cost?  
(Member vars that are arrays are inited by calling the default Ctor for each array element, if those Ctors acquire the resources then that acquisition can fail, how to return such a failure from the default Ctor?  
How and to who to return the failure from a Ctor of a global var? What happens if the global var Ctor throws?)
* In which cases is it _impossible_ to use the exceptions? Bare metal? (no OS to terminate the code that does not handle the exc)
* Some articles state that today's compilers generate such a code that the exceptions have _zero-cost in successful case_ (i.e. the exception is not thrown), but [Walter Bright](https://en.wikipedia.org/wiki/Walter_Bright) (creator of D) said (at http://nwcpp.org/ [November 2017 meeting](http://nwcpp.org/november-2017.html)) that _zero-cost exceptions are a myth_. Who is right? What particularly stands behind (what code is generated upon) `try`, `catch(T)`, `catch(...)`, `throw`, thrown-case, not-thrown-case, etc.? At C++Now2018's ["How Compilers Reason About Exceptions"](https://cppnow2018.sched.com/event/EC7V/how-compilers-reason-about-exceptions) session it has been recommended to read the ABI docs and the [[.eh_f]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md) Airs – Ian Lance Taylor. _".eh_frame"_ (still TODO).
* The Niall Douglas' talks about the exceptions, see the presentation [links](http://levelofindirection.com/refs/cpp-optional.html) recommended by Phil Nash during ["Option(al) Is Not a Failure"](https://cppnow2018.sched.com/event/EC7P/optional-is-not-a-failure) (TODO).

