The fragments of knowledge to support my notes during the code reviews (and the bookmarks for my own reference).

----
Nested (Local) Functions
-
[Walter Bright](https://en.wikipedia.org/wiki/Walter_Bright) (the creator of the D programming language) told during [November 2017 meeting](http://nwcpp.org/november-2017.html) of http://nwcpp.org/ how the nested (local) functions (that do not exist in C and C++) can help to avoid using `goto`. If I remember it right the approach looks like this:
```c++
// Not a valid C or C++ code:
bool enclosingFunc(..)  // A function that contains a nested function.
{
  char *c = NULL;       // Some resource pointers/descriptors/handles of the enclosingFunc().
  FILE *fileDescriptor = NULL;
  
  void nestedFunc(..)   // The nested function, is used to deallocate 
                        // the resources of the enclosingFunc().
  {                     // Has access to the enclosingFunc()'s local variables (c, fileDescriptor).
                        // Those resources are "global" for the nestedFunc().
    if(fileDescriptor != NULL)      // Deallocates the resources.
    {
      .. fclose(fileDescriptor)..
    }
    if(c != NULL)
    {
      delete[] c;
    }
  };
  
  // The regular code of the enclosingFunc():
  ..
  c = new char [..];  // Allocates a resource.
  ..
  if(..)
  {
    fileDescriptor = fopen(..);   // Allocates another resource.
    ..
        if(<someFailure>)
        {
          nestedFunc();  // Instead of `goto` deallocates the resources using the nested function,
          return false;  // returns in-place.
        }
        ..
            if(<someOtherFailure>)
            {
              nestedFunc(); // Instead of `goto` deallocates the resources using the nested function,
              return false; // returns in-place.
            }
  }
  
  return true;
}
```
The effect of the nested (local) functions can be simulated in C++ by using the following approaches.  

__1. The static function of the local class.__
```c++
bool enclosingFunc(..)
{ ..
  class localClass
  {
    static void nestedFunc(..) { .. }
  };
  
  // The regular code of the enclosingFunc().
}
```
Example:  
[[MC++D]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md), 11.6.2 The Logarithmic Dispatcher and Casts, p.281.  
Features:  
Very simple however the `localClass::nestedFunc()` has no access to the `enclosingFunc()`'s local variables (but the pointers/references to those can be passed to `nestedFunc()` as the arguments).  

__2. [[MExcC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md), Item 33: Simulating Nested Functions.__

Distinguish Between Size and Length
-
Sometimes I see the code similar to this:
```c++
char buffer[6]; // The buffer to read the chars to.
int bytesRead;  // The number of chars that have been read.
if((bytesRead = read(.., buffer, 5)) > 0)  // Read up to 5 chars to the buffer (instead of `5` 
    // there can be `sizeof(buffer) - sizeof((char)'\0')` but that's not the point).
{   // The `bytesRead` contains the number of chars actually read (1..5).
    buffer[bytesRead] = '\0';  // Null-terminate the sequence of chars in the buffer.
```
At some point in the future we can make a change like this:
```diff
< char buffer[6];
---
> wchar_t buffer[6];

```
(we replace `char` with `wchar_t`)  
Here, for simplicity, I assume that `wchar_t` is 2 bytes and `char` is 1 byte in size (but in reality their size is implementation-dependent).

After such a change we get problems:
* the call `read(.., buffer, 5)` (or `read(.., buffer, sizeof(buffer) - sizeof((char)'\0'))`) requests the _odd_ number of bytes, this can partially update one of the `whchar_t`s in the `buffer` (if the `read()` reads all 5 bytes (of the 5 requested) then the first 4 bytes will update the `buffer[0]` and `buffer[1]`, and the 5th byte will update the _half_ of the `buffer[2]`);
* The call `bytesRead = read(..)` updates the `bytesRead` variable with the _number of bytes_ (not the _number of characters_). But the subsequent fragment `buffer[bytesRead] = '\0'` requires the _index_ which in general case should be twice less than the _number of bytes_.
E.g. if the `bytesRead = read(..)` reads 4 bytes (of the 5 requested) and thus updates the `buffer[0]` and `buffer[1]` then we need to null-terminate the `buffer[2]` but the line `buffer[bytesRead] = '\0'` will null-terminate `buffer[4]`, and the `buffer[3]` will stay _UNinitialized_.

Based on similar observations I strictly distinguish between the _size_ and _length_.
* I use the concept of _size_ to designate the _size in bytes only_ (typically it is a result of the `sizeof()` operator), and I always prefer writing "size (in bytes)" rather than just "size".
* To designate the number of elements in a container/array, number of (char/wchar_t) characters in a string, I use the concept of _length_.
* The _length_ is always less than or equal to the _size_.
* The concept of _index_ (e.g. array index) originates from the concept of _length_ (but not from the concept of _size_).

E.g. for the declaration `wchar_t buffer[6]`
* the `buffer` _length_ is 6, the _index_ originates from _length_ and has a range from `0` to `(length - 1)` (from `0` to `5`);
* the `buffer` _size_ is at least 12 (and includes the optional alignment padding between (and probably before and after) the array elements).

The calls `read()`/`write()` expect as the last argument (and they return) _the number of bytes_ - a concept originating from _size_ (not from the _length_).
If we want to use _index_ as the last argument to `read()`/`write()` then the _index_ needs to be multipled by the size of the element (`index * sizeof(buffer[0])`)
and if we want to use the value returned by `read()`/`write()` to index the buffer then the value needs to be divided by the size of the element (`bytesRead / sizeof(buffer[0])`).
```c++
char /* or wchar_t */ buffer[6]; // The buffer to read the chars to.
int bytesRead;  // The number of bytes that have been read.
if((bytesRead = read(.., buffer,
                     sizeof(buffer) - sizeof(buffer[0]))) 
   > 0)
    // Read to the buffer up to 5 (char or wchar_t) characters
    // (up to 5 bytes for char or up to (>=10) bytes for wchar_t).
{   // The `bytesRead` contains the number of bytes actually read
    // (1..5 bytes for char, 1..(>=10) bytes for wchar_t).
    
    size_t index = bytesRead / sizeof(buffer[0]); // Calculate the index (to null-terminate).
    
    // Make sure the bytesRead is even for wchar_t case:
    // ASSERT((index * sizeof(buffer[0])) == bytesRead);
    
    buffer[index] = '\0';  // Null-terminate the sequence of
                           // (char or wchar_t) characters in the buffer.
```
----
The considered problem is easy to spot when replacing `char` with `wchar_t`. But it is harder to spot this problem when we port our code  
from the implementation where the `char` is 1 byte in size  
to the implementation where the `char` is 2 or more bytes in size (see below).

Some man pages (e.g. `man strncpy` in Ubuntu 12.04, 16.04, [this one](http://man7.org/linux/man-pages/man3/strncpy.3.html)) specify that the last argument of `strncpy()` is the number of _bytes_ (not the number of _characters_). Don't get mislead. It is the number of _characters_ ([[C99]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md) (_C11_): 7.21.2.4 (_7.24.2.4_) The `strncpy` function: _The `strncpy` function copies not more than `n`_ __characters__ ...). So write your code such that it still works if it is ported to the implementation where the `char` is 2 bytes in size.  
This man page error has been reported to the owners of [this](http://man7.org/linux/man-pages/man3/strncpy.3.html) man page on `2018.07.11`. The first reaction by Jakub was that _it is a common misconception that `char` type can have a size other than 1 byte_, I'm still workging on proofs pro or con.  

The `Clone()` member function (or Virtual Copy Constructor)
-
* [[MEC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md), Item 25: Virtualizing constructors and non-member functions.
* [[MExcC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md), _Item 31: Smart Pointer Members, Part 2: Toward a ValuePtr_
  + p.188 top, item (c);
  + p.193, first paragraph of section "Adding Extensibility Using Traits".

Know About the Compiler's Resource Allocation and Deallocation Order
-

__Conclusion__  
For the consistency with the compiler-generated code deinitialize the data in the reverse order of initialization.  

__Rationale__  
Let's say we have a class `D` derived from the base classes `B1` and `B2`. The `D` class also has member variables `m1` and `m2` of some classes `C1` and `C2`, and a default consructor:
```c++
class D : public B1, public B2
{
  C1 m1;
  C2 m2;
public:
  D();
};
```
Let's say we define the constructor like this:
```c++
D::D() :
  // Member Initialization List starts here:
  B2(),
  B1(),
  m2(),
  m1()
  // End of Member Initialization List.
{}
```
By default the g++ 4.6 (C++98/03) compiler will keep silent about the inconsistency between  
the order of calls in the programmer-written Member Initialization List and  
the code the compiler actually generates.

The order of calls in the actual compiler-generated code will be like this:
```c++
D::D() :
  // Member Initialization List starts here:
  B1(),
  B2(),
  m1(),
  m2(),
  // End of Member Initialization List.
{}
```
To get a warning/error about such an inconsistency one can use the compiler flags `-Wreorder`/`-Werror=reorder`.

The compiler orders the calls in the Member Initialization List (in all the constructors) to strictly correspond to the class definition (see the first-most listing in this topic) in order to guarantee the strict _REVERSE order of DEinitialization in the destructor_.

Confirmation in C++98:
> __12.6.2 Initializing bases and members__  
> 5:  Initialization shall proceed in the following order:  
— First, and only for the constructor of the most derived class as described below, virtual base classes shall be initialized in the order they appear on a depth-first left-to-right traversal of the directed acyclic graph of base classes, where “left-to-right” is the order of appearance of the base class names in the derived class _base-specifier-list_.  
— Then, direct base classes shall be initialized in declaration order as they appear in the _base-specifier-list_ (regardless of the order of the _mem-initializers_).  
— Then, nonstatic data members shall be initialized in the order they were declared in the class definition (again regardless of the order of the _mem-initializers_).  
— Finally, the body of the constructor is executed.  
\[_Note:_ __the declaration order is mandated to ensure that base and member subobjects are destroyed in the reverse order of initialization__. \]  

Also:  
> __12.6 Initialization__  
> 3: When an array of class objects is initialized (either explicitly or implicitly), the constructor shall be called for each element of the array, following the subscript order; see 8.3.4. \[_Note:_ __destructors for the array elements are called in reverse order of their construction__. \]  

> __12.4 Destructors__  
> 6: A destructor for class X calls the destructors for X’s direct members, the destructors for X’s direct base classes and, if X is the type of the most derived class (12.6.2), its destructor calls the destructors for X’s virtual base classes. .. __Bases and members are destroyed in the reverse order of the completion of their constructor__ (see 12.6.2). .. __Destructors for elements of an array are called in reverse order of their construction__ (see 12.6).  

> __5.3.5 Delete__  
> 6: The delete-expression will invoke the destructor (if any) for the object or the elements of the array being deleted. In the case of an array, the elements will be destroyed in order of decreasing address (that is, in reverse order of the completion of their constructor; see 12.6.2).

__See Also:__
* Search for "reverse order" in the C++ Standard.
* [g++ man page](http://man7.org/linux/man-pages/man1/gcc.1.html): `-Wreorder`/`-Werror=reorder`.

System Calls Failing with EINTR
-
_POSIX/Linux-specific_.  

A number of system calls upon failure return `-1` (or a negative value) and specify the reason of failre by setting the [`errno`](http://man7.org/linux/man-pages/man3/errno.3.html) to some value.

The `errno` value of [`EINTR`](http://man7.org/linux/man-pages/man3/errno.3.html) is a _special case_, it is not a failure as such, it can happen when _everything is correct_.  
E.g. our thread launches a child process (and continues the execution), then our thread calls [`select()`](http://man7.org/linux/man-pages/man2/select.2.html) with a 10-second time-out (and gets suspended), after 3 seconds the child process terminates (returns from its `main()`) which causes the [`SIGCHLD`](http://man7.org/linux/man-pages/man7/signal.7.html) signal to be sent to our thread, a signal handler is executed by the OS (within the context of our thread), the signal handler does nothing special (nothing related to `select()`), then the signal handler returns, and the suspended `select()` returns `-1` to our thread, with `errno` equal to `EINTR`.  
To summarize, our thread has called `select()` with a 10-second time-out, but the call has returned after 3 seconds; in general case this means that our thread should call `select()` again but this time with a 7-second time-out (and repeat the attempts as long as the `EINTR` happens, until the time-out is exhausted).  

Another example of when a system call can fail with the `EINTR` is when running a Linux console application in a terminal and re-sizing (or minimizing) the terminal window.

In general case I would handle the system call failures something like this:
```c++
do
{
    int retVal = select(..);   // The system call.
    int errNum = errno;        // Read `errno` immediately after the system call.

    if(retVal == -1)
    {
        switch(errNum) 
        {
            case EINTR:
                .. // Adjust the time-out.
                continue;
            case ENOMEM:
                WARN(..);
                .. // Failure Handling.
                return NULL;
            case EBADF:
            case EINVAL:
                ASSERT(.."Bug in our code! .." .. errNum ..);
                return NULL;
            default:
                ASSERT(.."Unknown or Unhandled Failure!" .. errNum ..);
                return NULL;
        }
    }
    else
        break;  // Exit from the `while()` loop.
}   
while(1);
```
__Info:__
* [`errno` Man Page](http://man7.org/linux/man-pages/man3/errno.3.html).
* [`select()` Man Page](http://man7.org/linux/man-pages/man2/select.2.html).
* [Alphabetic list of all Linux man pages](http://man7.org/linux/man-pages/dir_all_alphabetic.html).

Variable Length Arrays are C99 Feature, But Not C++
-
This item is applicable to (_strictly_) _Standard C++_ only.

Variable Length Arrays are the arrays whose length (number of elements) is a _run-time_ value (as opposed to _compile-time_ value). E.g.
```c++
void f(size_t runTimeValue)
{
  int stackBasedArray[runTimeValue];  // Variable Length Array.
```
By default the g++ (4.6) does not complain. But if we get pedantic (`-Wextra -pedantic`) then we get
* [ARM gcc 4.6.4 (linux) #1] warning: ISO C++ forbids variable length array .. [-Wvla]
* [x86-64 clang 5.0.0 #1] warning: variable length arrays are a C99 feature [-Wvla-extension]

Thus strictly speaking the Variable Length Arrays are not a C++ feature as of C++17 (but are C99 feature). The (strictly) Standard C++ code should not use them.

__Reasons:__  
One of the reasons why the Variable Length Arrays are not a C++ feature is the fact that C++ in general does not encourage the C-style arrays. Some other reasons are connected with uses in C++ different than in C, e.g. the arrays inside of classes, arrays of class instances, references to arrays, etc. See more details in the "See Also" section.

__Future:__  
There is an ongoing discussion about the alternatives for the Variable Length Arrays. See [[P0785R0] Runtime-sized arrays and a C++ wrapper](http://open-std.org/JTC1/SC22/WG21/docs/papers/2017/p0785r0.html), [[n3810] Alternatives for Array Extensions](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3810.pdf).


__See Also:__
* [C99], search for "variable length".
* [g++](http://man7.org/linux/man-pages/man1/gcc.1.html): `-Wvla`, `-Wvla-larger-than=n`.
* [[P0785R0] Runtime-sized arrays and a C++ wrapper](http://open-std.org/JTC1/SC22/WG21/docs/papers/2017/p0785r0.html).
* [[n3810] Alternatives for Array Extensions](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3810.pdf).

Know the Special Member Functions
-
__Story:__  
* C++98/03: [[EC++3]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md) Chapter 2: Constructors, Destructors, and Assignment Operators.
* C++11: [What the Move Semantics Are](https://stackoverflow.com/questions/3106110/what-are-move-semantics) (see both replies by `fredoverflow`).
* C++11: [[EMC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md) Item 17: Understand special member function generation.

__Consequences:__
* [Know All the Effects of the Empty Destructor](https://github.com/kuzminrobin/code_review_notes/blob/master/cpp_design_bookmarks.md#know-all-the-effects-of-the-empty-destructor).
* [There Should Be a Strong Reason for Writing the Destructor](https://github.com/kuzminrobin/code_review_notes/blob/master/cpp_design_bookmarks.md#there-should-be-a-strong-reason-for-writing-the-destructor).
* [Know the Peculiarities of Writing the Assignment Operator](https://github.com/kuzminrobin/code_review_notes/blob/master/cpp_design_bookmarks.md#know-the-peculiarities-of-writing-the-assignment-operator).

Know All the Effects of the Empty Destructor
-
Some people were taught to always write the destructor just in case, even if it does nothing (is empty).

Starting in C++11 the explicit destructor 
* disables the (implicit, by-compiler) generation of the move operations,
* makes the generation of the copy opertions _deprecated_ (i.e. _disabled_ in some future version of C++).

See the end (_Things to Remember_ section) of [[EMC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md) Item 17: Understand special member function generation, fragments:
* Move operations are generated only for classes lacking explicitly declared move operations, copy operations, and a _destructor_.
* Generation of the copy operations in classes with an explicitly declared _destructor_ is deprecated.

__Broader Picture:__  
* [Know the Special Member Functions](https://github.com/kuzminrobin/code_review_notes/blob/master/cpp_design_bookmarks.md#know-the-special-member-functions).
* Pimpl-Idiom-specific, when the empty destructor may be needed: [[MExcC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md):
  + _Item 30: Smart Pointer Members, Part 1: A Problem with `auto_ptr`_, section "What About `auto_ptr` Memebers?", last but one paragraph on page 185: "If you don't want to provide the definition of Y, you must write the X2 destructor explicitly, even if it's empty".
  + _Item 31: Smart Pointer Members, Part 2: Toward a `ValuePtr`_, section "A Simple `ValuePtr`: Strict Ownershipt Only", first regular paragraph: "Either the full definition of Y must accompany X, or the X destructor must be explicitly provided, even if it's empty".

There Should Be a Strong Reason for Writing the Destructor
-
If you use Smart Pointers and other Smart Resource Releasers (see below) then you don't need the explicit (i.e. manually written) destructor, the implicit (i.e. the compiler-generated) one will do the right thing in the [right order](https://github.com/kuzminrobin/code_review_notes/blob/master/cpp_design_bookmarks.md#know-about-the-compilers-resource-allocation-and-deallocation-order).  

_The explicit destructor is a warning sign._

__Broader Picture:__  
* [Know the Special Member Functions](https://github.com/kuzminrobin/code_review_notes/blob/master/cpp_design_bookmarks.md#know-the-special-member-functions).
* Smart Pointers:
  * C++98/03: [[EC++3]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md)
    * Chapter 3: Resource Management
    * Item 29: Strive for exception-safe code
    * Item 45: Use member function templates to accept "all compatible types."
    * Search for "smart" in the book
  * C++98/03: [[MEC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md) _Item 28: Smart pointers_ and other items referred to in Item 28.
  * C++11/14: [[EMC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md) Chapter 4. Smart Pointers
* Smart Resource Releasers: C++98/03: [[gcwywescf]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md).

Know the Peculiarities of Writing the Assignment Operator
-
When writing the _Copy-Assignment Operator_ the following logic needs to be applied.
* Know the [Special Member Functions](https://github.com/kuzminrobin/code_review_notes/blob/master/cpp_design_bookmarks.md#know-the-special-member-functions) (in particular [[EC++3]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md) Item 5: Know what functions C++ silently writes and calls, [[EMC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md) Item 17: Understand special member function generation) and _try to avoid writing the explicit assignment operator_.
* If you have to write then _try to avoid the [Copy-And-Swap Idiom](https://stackoverflow.com/a/3279550/6362941)_ (e.g. if your Assignment Operator does not make throwing calls) since the idiom creates a copy thus lowering down the performance.
* Use the [Copy-And-Swap Idiom](https://stackoverflow.com/a/3279550/6362941) _as the last resort_.
* Don't forget to _call the parent class' Assignment Operator_, see [[EC++3]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md) Item 12: Copy all parts of an object.

TODO: Review for the _Move-Assignment Operator_.  

__Broader Picture:__  
* [Know the Special Member Functions](https://github.com/kuzminrobin/code_review_notes/blob/master/cpp_design_bookmarks.md#know-the-special-member-functions).

Inlining
-
[[EC++3]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md):
* Item 2: Prefer consts, enums, and inlines to #defines.
* Item 5: Know what functions C++ silently writes and calls, fragment: "_All these functions will be both public and inline_".
* Item 30: Understand the ins and outs of inlining.
* Item 44: Factor parameter-independent code out of templates, p. 214(paper)/235(file), Fragment: "_(Provided, of course, you refrain from declaring that function inline. If it’s inlined, each instantiation of SquareMatrix::invert will get a copy of SquareMatrixBase::invert’s code (see Item 30), and you’ll find yourself back in the land of object code replication.)_".
* Also search for `inline`, `inlining` in the file.

[[MEC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md):
* Item 24: Understand the costs of virtual functions, multiple inheritance, virtual base classes, and RTTI; p.118(paper)/135(file), middle, paragraph starting with: "The real runtime cost of virtual functions has to do with their interaction with inlining. For all practical purposes, virtual functions aren’t inlined.".
* TODO: Other `inline`, `inlining` occurrences in Item 26, and other places.

TODO: [ExcC++].

To be continued.

Know the Danger of Overflowing (and Underflowing) the Signed Types
-
The _signed_ and _unsigned_ integer types behave differently when overflown or underflown.  

First let's consider by the example of the _unsigned_ types.  
If we have an _unsigned_ 8-bit integer (whose range is from 0 (0x00) to 255 (0xFF)) with value 0, and we decrement it (thus causing the underflow), then we get the value 255 (0xFF). Fully defined value and fully defined behavior.
In the same way incrementing the value 255 (0xFF) (thus causing the overflow) will result in the value 0 (0x00). Also a fully defined value and fully defined behavior.  

Confirmation in C++98:  
> __3.9.1 Fundamental types__  
> 4: Unsigned integers, declared unsigned, shall obey the laws of arithmetic modulo 2^n where n is the number of bits in the value representation of that particular size of integer. Footnote 41)  
> 41) This implies that unsigned arithmetic does not overflow because a result that cannot be represented by the resulting unsigned integer type is reduced modulo the number that is one greater than the largest value that can be represented by the resulting unsigned integer type.

But for the _signed_ integers the picture is different.  
If we have a signed 8-bit integer, e.g. `int8_t` (with the range  
from `std::numeric_limits<int8_t>::min()` in C++, or `INT8_MIN` in C,  
to `std::numeric_limits<int8_t>::max()` in C++, or `INT8_MAX` in C),  
with the lowest value `std::numeric_limits<int8_t>::min()` (`INT8_MIN`) and we decrement such a value then we can get the _Undefined Behavior_! (Here and below, for brevity and simplicity, the integer promotion rules are ignored, i.e. it is assumed that the 8-bit value is _not_ extended to 16-bit or 32-bit value then decremented _without the overflow_ and then the least significant byte is saved back to the 8-bit integer, all in _fully defined_ way. We assume that any sigend integer type (`int8_t`, `int16_t`, etc.) and/or all of those types can be implemented using `int` (or using a type larger than `int`) on some implementations)  
The same is for incrementing the highest value `std::numeric_limits<int8_t>::max()` (`INT8_MAX`).  
It is emphasized, not just the _value_ is undefined, but the whole _behavior_ is undefined.  

The same is for the floating point types.  

C++98:  
> __5 Expressions__  
5: If during the evaluation of an expression, the result is not mathematically defined or not in the range of representable values for its type, the behavior is undefined, unless such an expression is a constant expression (5.19), in which case the program is ill-formed.

__For the portable code this all means the following:__  

All the calculations that rely on the integer type overflowing, e.g. the checksums (CRC), hash codes, ring buffer indices, must use the _unsigned_ types (this is also applicable to all the unsigned concepts such as size in bytes, length of the sequence (e.g. the number of elements in an array), array/container index, offset, pointer arithmetic, bitmasks (and shifts for them)).

Negating the signed value (`-signedValue`) and  
  + assigning such a value to oneself (`x = -x;`) or to a variable of the same type (`y = -x;`)
  + or passing such a value as an argument (corresponding to the parameter of the same type), e.g.  
    `void foo(int y); main() { int x =..; foo(-x); }`  

can cause overflow, i.e. Undefined Behavior, if before the negation the value was `std::numeric_limits<int??_t>::min()` (`INT??_MIN`). This is because negating such a value in some representations (see below) can result in a value (`std::numeric_limits<int??_t>::max()` + 1) (`INT??_MIN` + 1) which does not fit in the type.  

__Reasons:__  

One of the reasons why unsigend and signed integers behave so differently is the fact that the signed integers have at least [3 implementation-dependent representations](https://softwareengineering.stackexchange.com/questions/239036/how-are-negative-signed-values-stored/239039#239039) in C.  

__Future:__  

The Proposals [to C](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2218.htm) and [C++](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0907r1.html) named "Signed Integers are Two’s Complement" tell that there is an intention to use in C and C++ only one representation for the signed integers (which allows applying the same algorithms and/or hardware logic both for signed and unsigned integer arithmetic (at least addition and subtraction, and very likely multiplication and division)). After those Proposals end up in the C and C++ Standards the overflow and underflow of the signed integers are expected to become fully defined (the same way as for the unsigned integers).  
The same fully defined effect for signed integers in [gcc/g++](http://man7.org/linux/man-pages/man1/gcc.1.html) can be achieved _now_ by using the `-fwrapv` command line argument (but the code relying on such a wrapping behavior is considered _non-portable_ as of C11 and C++17).  

__More Info:__  
* C++Now 2018 Closing Keynote: Undefined Behavior and Compiler Optimizations, _John Regehr_ ([abstract](http://sched.co/ELaF), [slides](https://github.com/boostcon/cppnow_presentations_2018/blob/master/05-11-2018_friday/undefined_behavior_and_compiler_optimizations__john_regehr__cppnow__05112018.pdf), [video](https://youtu.be/AeEwxtEOgH0)).  
* C++Now 2014 Undefined Behavior in C++: What is it, and why do you care? _Marshall Clow_ ([video](https://www.youtube.com/watch?v=uHCLkb1vKaY)).  
* [gcc/g++](http://man7.org/linux/man-pages/man1/gcc.1.html) command line arguments: -fstrict-overflow, -Wstrict-overflow[=n], -Wno-overflow, -fwrapv, -fsanitize=signed-integer-overflow, -fsanitize=float-cast-overflow, -ftrapv, also search for "overflow".

Info Sources About the Exceptions
-
Partially based on Deb Haldar's _Top 15 C++ Exception handling mistakes and how to avoid them_ [[t15c++ehm]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md), _Where do we go from here?_ section.

__Books and Articles at a High Level__
* `(Ongoing )` [C++ Exception FAQ](https://isocpp.org/wiki/faq/exceptions) on isocpp.org (still TODO).
* `1996.??.??` [More Effective C++ – 35 new ways to improve your programs and designs](https://www.amazon.com/More-Effective-Improve-Programs-Designs/dp/020163371X/ref=sr_1_1?ie=UTF8&qid=1470238686&sr=8-1&keywords=more+effective+c+meyers) – items 9 through 15 [[MEC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md).
* `1999.11.18` [Exceptional C++ – 47 Engineering puzzles, programming problems and solutions](https://www.amazon.com/Exceptional-Engineering-Programming-Problems-Solutions/dp/0201615622/ref=sr_1_1?ie=UTF8&qid=1470238861&sr=8-1&keywords=Exceptional+C%2B%2B) – items 8 through 19 [[ExcC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md).
* `2001.12.17` [[MExcC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md), Chapter "Exception Safety Issues and Techniques", Items 17 - 23.
* `2004.10.??` [C++ Coding Standards – 101 Rules, Guidelines and Best Practices](https://www.amazon.com/Coding-Standards-Rules-Guidelines-Practices/dp/0321113586/ref=sr_1_1?ie=UTF8&qid=1470238746&sr=8-1&keywords=C%2B%2B+Coding+standards) – items 68 through 75 [[C++CS]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md).
* `2015.??.??` [[e&su]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md) MSDN. _Exceptions and Stack Unwinding in C++_.
* `2016.08.03` [[t15c++ehm]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md) Deb Haldar. Top 15 C++ Exception handling mistakes and how to avoid them.

__Q&As__
* The [Copy-And-Swap Idiom](https://stackoverflow.com/a/3279550/6362941).

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

