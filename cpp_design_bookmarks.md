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
* [[t15c++ehm]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md)
