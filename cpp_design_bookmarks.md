C++ Exception Specifications
-

The Herb Sutter's article ["A Pragmatic Look at Exception Specifications"](http://www.gotw.ca/publications/mill22.htm) (July 2002) makes me think that the exception specifications  
* instead of providing the compile-time no-throw-guarantee (`throw()`) or throws-only-these-ones-gaurantee (`throw(A, B)`)  
* provide the run-time checks/asserts (which are _pessimization_, i.e. they lower down the performance).  

The Meyers' [[EMC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md), Item 14: Declare functions `noexcept` if they won’t emit exceptions  
in the beginning makes me think that `noexcept` helps to optimize in certain cases (by means of replacing the copy operations with the move operations). But in the end still shows that there are no any copile-time guarantees (a `noexcept`-function can call the functions without the exception specification, and the compilers (so far) don't complain) but there are run-time checks ("_your program will be terminated if an exception tries to leave the function_").
