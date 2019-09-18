# The Code-Reviewer's Notes
[About the author and how to get in touch](https://docs.google.com/document/d/1TmA8cXRxb_9lDnCREfYuIeGpJy9sMsXSpFVMK-0PwNo/edit).  

The unsorted fragments of knowledge to support my notes during the code reviews (and the bookmarks for my own reference).  

----
+ [Identifiers](#identifiers)
+ [Booleans](#booleans)
  + [Prefer `bool` For Booleans](#prefer-bool-for-booleans)
  + [Avoid Comparing Booleans to `true`](#avoid-comparing-booleans-to-true)
+ [The `memset()` Function Is a Warning Sign](#the-memset-function-is-a-warning-sign)
  + [Know the Limitations of `memset()` When Initializing](#know-the-limitations-of-memset-when-initializing)
  + [Andrey Karpov's Experience With `memset()`](#andrey-karpovs-experience-with-memset)
  + [How to Automate Catching the `memset()` Problems](#how-to-automate-catching-the-memset-problems)
  + [How to Force the `memset()` Problems to Show Themselves](#how-to-force-the-memset-problems-to-show-themselves)
+ [Comparing Floating Point Numbers For [In]Equality](#comparing-floating-point-numbers-for-inequality)
+ [Overload Resolution](#overload-resolution)
+ [const T vs. T const (aka `const West` vs. `East const`)](#const-t-vs-t-const-aka-const-west-vs-east-const)
+ [Know the Danger of `printf()` and Similar Functions](#know-the-danger-of-printf-and-similar-functions)
+ [Nested (Local) Functions](#nested-local-functions)
+ [Distinguish Between Size and Length](#distinguish-between-size-and-length)
+ [The `Clone()` member function (or Virtual Copy Constructor)](#the-clone-member-function-or-virtual-copy-constructor)
+ [Know About the Compiler's Resource Allocation and Deallocation Order](#know-about-the-compilers-resource-allocation-and-deallocation-order)
+ [System Calls Failing with `EINTR`](#system-calls-failing-with-eintr)
+ [Variable Length Arrays are C99 Feature, But Not C++](#variable-length-arrays-are-c99-feature-but-not-c)
+ [Know the Danger of `alloca()`](#know-the-danger-of-alloca)
+ [Know the Special Member Functions](#know-the-special-member-functions)
  + [Know All the Effects of the Empty Destructor](#know-all-the-effects-of-the-empty-destructor)
  + [There Should Be a Strong Reason for Writing the Destructor](#there-should-be-a-strong-reason-for-writing-the-destructor)
  + [Polymorphic Behavior of Non-Virtual Destructor](#polymorphic-behavior-of-non-virtual-destructor)
  + [Know the Peculiarities of Writing the Assignment Operator](#know-the-peculiarities-of-writing-the-assignment-operator)
+ [Inlining](#inlining)
+ [Know the Danger of Overflowing (and Underflowing) the Signed Types](#know-the-danger-of-overflowing-and-underflowing-the-signed-types)
+ [Info Sources About the Exceptions](#info-sources-about-the-exceptions)
  + [Certain Code Fragments Should Not Throw Exceptions](#certain-code-fragments-should-not-throw-exceptions)
  + [C++ Exception Specifications](#c-exception-specifications)
  + [C++ Exceptions: TODO](#c-exceptions-todo)
+ [Curious Fragments and Questions](#curious-fragments-and-questions)
  + [Abstract Class Constructors: `public`? `private`?](#abstract-class-constructors-public-private)

----
## Identifiers

Some categories of identifiers are reserved by the C++ standard. See  
[[C++98]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#Cpp98) 17.4.3.1.2 Global names,  
[C++17_N4659] 5.10 Identiﬁers, paragraphs 3.1 and 3.2 
> - Each identiﬁer that contains a double underscore `__` or begins with an underscore followed by an uppercase letter is reserved to the implementation for any use.  
> - Each identiﬁer that begins with an underscore is reserved to the implementation for use as a name in the global namespace. 

If your code uses such identifiers (e.g. `__MY_HEADER_H_INCLUDED`, `__my_local_var`, `_MyMemberVar`, `_myGlobalVar`) then upon next edition of the standard (i.e. upon migration to the next version of your compiler) your code can start behaving in an unexpected/unpredictable way.

## Booleans

### Prefer `bool` For Booleans
The `bool` type has not been included in the [C89]/[C90] standard. For a number of years prior to that and for almost a decade after that all the C and C++ programmers were using their own implementation of the Boolean type. Then starting with [C++98] the `bool` type has been standardized in C++. Shortly after that starting with [C99] it has been standardized in C. But having a large code base and a strong habit people were still using their own implementation. The situation was also facilitated by the fact that not all the compilers were supporting `bool`, e.g. Visual Studio started supporting `bool` (to be more precise, started providing [<stdbool.h>](http://www.cplusplus.com/reference/cstdbool/?kw=stdbool.h)) in C with the version of 2015 (or 2012, double-check). In such a situation of having `bool` in C++ and not having in C for a long time, and partially for code portability between C++ and C, many still are using own Boolean implementations.  
Although `bool` type has been standardized both in C and C++ for nearly two decades now, in C it still requires including [<stdbool.h>](https://en.cppreference.com/w/c/types/boolean) whereas in C++ it is a language keyword and does not require anything extra. Thus the support is still suboptimal.
None the less the [type `bool`](https://en.cppreference.com/w/cpp/keyword/bool) (and its values `false` and `true`) still has its advantages over the other alternatives of Boolean.
* The type `bool` is standard both in C and C++. It does not cause any questions for those who know about it. Whereas such types as `BOOL` or `Bool` or other Boolean implementations cause questions (e.g. "What is the difference of this type from `bool`?") and typically force to take a look at the implementation in order to understand the difference from `bool` and limitations originating from that.  
* When converting a number of types to `bool` the [Boolean conversions](https://en.cppreference.com/w/cpp/language/implicit_conversion#Boolean_conversions) guarantee that the `bool` will take one of the two values - either `false` or `true`, whereas converting to other implementations can result in more than two values, e.g. converting value `5` to some implementation, let’s say named `BOOL`, can result in a value not equal to `FALSE` and not equal to `TRUE`. If that implementation is based on the _unsigned_ integer type then the behavior will still be defined. But if the implementation is based on the _signed_ integer type then the behavior can become _undefined_ if the conversion [overflows that signed interger type](#know-the-danger-of-overflowing-and-underflowing-the-signed-types). Also (as [Alexander Zaitsev](https://github.com/zamazan4ik) has informed) if the implementation is the enum type, e.g. `typedef enum { FALSE, TRUE } BOOL;` then the resulting value not equal to `FALSE` and not equal to `TRUE` is _unspecified_ until [C++17] and causes the _undefined behavior_ since [C++17] (search for "unspecified" in the [Enumeration declaration](https://en.cppreference.com/w/cpp/language/enum)).  
[Andrey Karpov]() has also admitted, the conversion of any (raw) pointer to `bool` is consistent (any non-null pointer is converted to `true`) whereas the conversion to other implementations can be _inconsistent_, e.g. converting non-null 16-bit pointer with 8 least significant bits equal to 0 (e.g. 0xAB00) to a 8-bit Boolean implementation can result in a value of 0 equivalent to `false` (i.e.   
converting the non-null pointer `0xAB00` results in the equivalent of `false` (`0x00`), but  
converting the non-null pointer `0xAB01` results in the equivalent of `true`  (`0x01`)).
* If you use some type (e.g. `int`) for your Boolean implementation then you will not be able to provide two distinct function overloads, one for that type (`int`) and one for Boolean, e.g. like this
```c++
void f(int);
void f(BOOL);
```

#### __See Also:__
* [V721](https://www.viva64.com/en/w/v721/). The VARIANT_BOOL type is utilized incorrectly. The true value (VARIANT_TRUE) is defined as -1 (+[RU](https://www.viva64.com/ru/w/v721/)).  

#### __What to Remember:__  
For Booleans prefer using the type `bool` and its values `false` and `true`.

#### __How to Automate Catching This:__  
Help is wanted...  

__[PVS-Studio](https://www.viva64.com/en/pvs-studio/):__  
* [V724](https://www.viva64.com/en/w/v724/). Converting integers or pointers to BOOL can lead to a loss of high-order bits. Non-zero value can become 'FALSE' (+[RU](https://www.viva64.com/ru/w/v724/)).  

#### __How to Force the Bugs to Show Themselves:__  
Help is wanted...  

### Avoid Comparing Booleans to `true`

Comparing a Boolean value to `true` is unreliable.

Sometimes there are cases when the integer value `0` acts as an indication of `false`. And an arbitrary non-zero integer value acts as an indication of `true` (e.g. such a value can be returned by a function). Comparing the (`true`-like) non-zero integer value to `true` (or to the user-defined constant `TRUE`) is likely to fail.

If such an integer value (with Boolean-like behavior) is assigned to a `bool` variable then the [Boolean conversions](https://en.cppreference.com/w/cpp/language/implicit_conversion#Boolean_conversions) guarantee that the `bool` will still stay `false` (0) or `true` (1). However the programmers sometimes do such tricks that the unexpected value still penetrates into the `bool` variable. E.g. they copy the Boolean-like integer byte-by-byte into the `bool`. This results in an arbitrary non-zero value, acting as `true`, to be in the `bool` variable instead of `true`.  

The section [Know the Limitations of `memset()` When Initializing](#know-the-limitations-of-memset-when-initializing) shows a particular example of how `bool` can get an unexpected value.

__What to remember:__  
Avoid comparing Booleans to `true`.  
Prefer comparing to `false` (`== false`, `!= false`) or comparing like this: `if(boolVar)`, `if( ! boolVar)`.

__How to automate catching this:__  
(To some extent) [V676. It is incorrect to compare the variable of BOOL type with TRUE](https://www.viva64.com/en/w/v676/) (+[RU](https://www.viva64.com/ru/w/v676/)).  
See also the [How to Automate Catching the `memset()` Problems](#how-to-automate-catching-the-memset-problems) section.

__How to force the bugs caused by this to show themselves:__  
In progress... Help is welcome.  
See also the [How to Force the `memset()` Problems to Show Themselves](#how-to-force-the-memset-problems-to-show-themselves) section.

## The `memset()` Function Is a Warning Sign
This section describes the problems with the `memset()` function.

### Know the Limitations of `memset()` When Initializing
The problem can show itself in the code like this:
```c++
typedef enum
{
    ID_A,
    ..
    ID_D,
    ID_IDLE  // Let's say this value is 4.
}
ENUM_TYPE;

ENUM_TYPE myEnumArray[2]; // Array of enumerations.
bool myBoolArray[3];      // Array of Booleans.

// Initialize all the items of `myEnumArray` to `ID_IDLE`:
memset(myEnumArray, ID_IDLE, sizeof(myEnumArray));

// Initialize all the items of `myBoolArray` to `true`:
memset(myBoolArray, true, sizeof(myBoolArray));
```
Recall that [`memset()`](https://en.cppreference.com/mwiki/index.php?title=Special%3ASearch&search=memset) converts its second parameter to `unsigned char` and copies that value to the bytes starting with the address pointed to by the first parameter.

Converting to `unsigned char` can be problematic if the value of `ID_IDLE` does not fit in the `unsigned char`. With years the code can grow such that the `ID_IDLE` becomes `0x100`. After converting to `unsigned char` it becomes `0x00` and all the items of `myEnumArray` instead of being intialized to `ID_IDLE` start being initialized to `ID_A`.

There is one more problem with `memset()`.  
If in the code above the `sizeof(ENUM_TYPE)` is `1` and `sizeof(bool)` is `1` then we are fine. However the `sizeof(ENUM_TYPE)` is [not guaranteed](https://stackoverflow.com/a/8115895/6362941). The `sizeof(bool)` is [not guaranteed](https://en.cppreference.com/w/cpp/language/types) either. That is why if the compiler decides to use 2 bytes for each of those types above then  
the items of `myEnumArray` will be initialized not to the value of `ID_IDLE` (`0x04`) but to the value of `0x0404`,  
the items of `myBoolArray` will be initialized not to the value of `true` (`0x01`) but to the value of `0x0101`.  
For such values (`0x0101`) of `myBoolArray`  
if your code compares those with `false` (`== false`, `!= false`) or evaluates like this: `if(myBoolArray[0])`,  `if( ! myBoolArray[0])` then the code will be behaving as expected,  
but if your code compares those with `true` (`== true`, `!= true`) then the comparison is likely to _fail_. That is one of the reasons why I recommend to [_avoid comparing Booleans to `true`_](#avoid-comparing-booleans-to-true).

__What to remember:__  

If you need to initialize the multi-byte varaibles (or the variables whose size is not guaranteed), or arrays of those, or structures containig those, etc.  
to the value not containing `0` in all the bit positions,  
and not containing `1` in all the bit positions,  
then applying `memset()` for such an initialization is a risk.  

In other words you can safely use `memset()` to initialize  
1-byte-long variables (or types containing those)  
or multi-byte variables to `0` in all bit positions  
or multi-byte variables to `1` in all bit positions.  
In all other cases applying `memset()` is unsafe.

### Andrey Karpov's Experience With `memset()`
In his posts [Andrey](https://www.viva64.com/en/b/a/andrey-karpov/) has described many more problems with `memset()` function. The number and severity of the problems really impresses. Please see his posts:  
* [The most dangerous function in the C/C++ world](https://www.viva64.com/en/b/0360/) (+[RU](https://www.viva64.com/ru/b/0360/))  
* [Safe Clearing of Private Data](https://www.viva64.com/en/b/0388/) (+[RU](https://www.viva64.com/ru/b/0388/))  

**See also:**  
[CWE-14: Compiler Removal of Code to Clear Buffers](https://cwe.mitre.org/data/definitions/14.html)

### How to Automate Catching the `memset()` Problems
In progress...  

__[PVS-Studio](https://www.viva64.com/en/pvs-studio/):__  
The following diagnostics catch some of the `memset()` problems.
* [V575 The 'memset' function processes value XXXX](https://www.viva64.com/en/w/v575) (+[RU](https://www.viva64.com/ru/w/v575))  
Catches the poblem of narrowing conversion of the second argument to `unsigned char` (when the value of `0x100` becomes `0x00`).
* [V601 The 'true' value is implicitly cast to the integer type](https://www.viva64.com/en/w/v601/) (+[RU](https://www.viva64.com/en/w/v601/))  
Catches the poblem of implicit cast  of the `bool` second argument to `unsigned char`.
* [V597 The compiler could delete the 'memset'...](https://www.viva64.com/en/w/v597/) (+[RU](https://www.viva64.com/ru/w/v597/))  
Catches the security problem when the compiler could delete the `memset()` function call.
* The diagnostic catching the problem  
when the items of `myEnumArray` get initialized not to the value of `ID_IDLE` (`0x04`) but to the value of `0x0404`  
has been added to the PVS-Studio's _To Do_ list (on 2019.03.0x). We are looking forward to the new diagnostic.

### How to Force the `memset()` Problems to Show Themselves
In progress...  
The problem described in section [Know the Limitations of `memset()` When Initializing](#know-the-limitations-of-memset-when-initializing) is likely to cause different behavior  
* at different optimization levels of the same compiler (run the same tests with no optimization, highest optimization for speed, highest optimization for size),  
* on architectures with different native size (especially 8-bit vs. 16-bit).

Visual Studio 2017 Enterprise x64 (19.16.27027.1 for x86) (`cl.exe /EHsc <file.cpp>`):  
`std::cout << "myEnumArray[0]: 0x" << std::hex << myEnumArray[0] << "\n";`  
`myEnumArray[0]: 0x4040404`.  

IAR C/C++ Compiler V6.70.2.6274/W32 for ARM (`arm\bin\iccarm.exe`):  
`--enum_is_int`   `Force the size of all enumeration types to be at least 4 bytes`.  

Comparing Floating Point Numbers For [In]Equality
-
Here is what one needs to know before comparing the floating point numbers for [in]equality.
* GCC's `-Wfloat-equal` option on [Options to Request or Suppress Warnings](https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html) page or [gcc man page](http://man7.org/linux/man-pages/man1/gcc.1.html).
* [[cfpn]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#cfpn) _Comparing Floating Point Numbers, 2012 Edition_ by brucedawson.

Overload Resolution
-
`2018.??.??` [[C++TCG]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#C++TCG) David Vandevoorde, Nicolai M. Josuttis, Douglas Gregor. _C++ Templates: The Complete Guide_ (2nd Edition), _Appendix C: Overload Resolution_, paper pages 681 - 696.

const T vs. T const (aka `const West` vs. `East const`)
-
__Story, Grounds, and Mechanism:__  
`1999.02.??` [[ctvtc]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#ctvtc) Dan Saks. _const T vs. T const_. Embedded Systems Programming, FEBRUARY 1999.  
`1998.06.??` [[pcd]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#pcd) Dan Saks. _Placing `const` in Declarations_. Embedded Systems Programming, JUNE 1998.  

__Where It Can Matter:__  
`2018.??.??` [[C++TCG]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#C++TCG) David Vandevoorde, Nicolai M. Josuttis, Douglas Gregor. _C++ Templates: The Complete Guide_ (2nd Edition), section _Some Remarks About Programming Style_, paper pages _xxxi - xxxii_.  
`2011.09.26` [[scs]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#scs) Dan Saks. _Simplifying `const` Syntax_. Dr.Dobb's, September 26, 2011.  

__Videos:__  
(About `East const` vs. `const West`, mostly _for fun_)  
* `East const`: [C++Now 2018: Jon Kalb "This is Why We Can't Have Nice Things"](https://www.youtube.com/watch?v=fovPSk8ixK4) (5:29).
* `const West`: [C++Now 2018: Jonathan Müller "A Fool's Consistency"](https://www.youtube.com/watch?v=_27NHB1OlNI) (4:21).


Know the Danger of `printf()` and Similar Functions
-
How hackers exploit `printf()`:  
`2012.02.01` [[wnuw]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#wnuw) PVS Articles: Andrey Karpov. _Wade not in unknown waters. Part two_.  

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
[[MC++D]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#MC++D), 11.6.2 The Logarithmic Dispatcher and Casts, p.281.  
Features:  
Very simple however the `localClass::nestedFunc()` has no access to the `enclosingFunc()`'s local variables (but the pointers/references to those can be passed to `nestedFunc()` as the arguments).  

__2. [[MExcC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#MExcC++), Item 33: Simulating Nested Functions.__

Distinguish Between Size and Length
-
Sometimes I see the code similar to this:
```c++
#include <unistd.h>  // Declares `read()`.

char buffer[6]; // The buffer to read the chars to.
int bytesRead;  // The number of chars that have been read.
if((bytesRead = read(.., buffer, 5)) > 0)  // Read up to 5 bytes to the buffer (instead of `5` 
    // there can be `sizeof(buffer) - sizeof((char)'\0')` but that's not the point).
{   // The `bytesRead` contains the number of bytes actually read (1..5).
    buffer[bytesRead] = '\0';  // Null-terminate the sequence of chars in the buffer.
```
(Here is the `read()` [man page](http://man7.org/linux/man-pages/man2/read.2.html))

At some point in the future we can make a change like this:
```diff
< char buffer[6];
---
> wchar_t buffer[6];

```
(we replace `char` with `wchar_t`)  
Here, for simplicity, I assume that  
`wchar_t` is 2 bytes in size (but in reality its size is implementation-defined, see below),  
whereas `char` is 1 byte in size,  
see
* C++11 Late Working Paper n3242, 5.3.3 Sizeof, item 1, fragments 
  * "`sizeof(char)`, `sizeof(signed char)` and `sizeof(unsigned char)` are 1",
  * "in particular, `sizeof(..)`, .. , and `sizeof(wchar_t)` are implementation-defined".
* C11 (N1570) 6.5.3.4 The `sizeof` and `_Alignof` operators, item 4, fragment "When `sizeof` is applied to an operand that has type `char`, `unsigned char`, or `signed char`, (or a qualified version thereof) the result is 1").

After such a change we get problems:
* the call `read(.., buffer, 5)` (or `read(.., buffer, sizeof(buffer) - sizeof((char)'\0'))`) requests the _odd_ number of bytes, this can _partially_ (incompletely) update one of the `wchar_t`s in the `buffer` (if the `read()` reads all 5 bytes (of the 5 requested) then the first 4 bytes will update the `buffer[0]` and `buffer[1]`, and the 5th byte will update the _half_ of the `buffer[2]`);
* The call `bytesRead = read(..)` updates the `bytesRead` variable with the _number of bytes_ (not the _number of characters_). But the subsequent fragment `buffer[bytesRead] = '\0'` requires the _index_ which (for our `wchar_t`) should be twice less than the _number of bytes_.
E.g. if the `bytesRead = read(..)` reads 4 bytes (of the 5 requested) and thus updates the `buffer[0]` and `buffer[1]` then we need to null-terminate the `buffer[2]` but the line `buffer[bytesRead] = '\0'` will null-terminate `buffer[4]`, and the `buffer[3]` will stay _UNinitialized_.

Based on similar observations I strictly distinguish between the _size_ and _length_.
* I use the concept of _size_ to designate the _size in bytes only_ (typically it is a result of the `sizeof()` operator), and I always prefer writing "size (in bytes)" rather than just "size".
* To designate the number of elements in a container/array, number of (char/wchar_t) characters in a string, I use the concept of _length_.
* The _length_ is always less than or equal to the _size_ (in bytes).
* The concept of _index_ (e.g. array index) originates from the concept of _length_ (but not from the concept of _size_).

E.g. for the declaration `wchar_t buffer[6]`
* the `buffer` _length_ is 6, the _index_ originates from _length_ and has a range from `0` to `(length - 1)` (from `0` to `5`);
* the `buffer` _size (in bytes)_ is at least 12 (and includes the optional alignment padding between (and probably before and after) the array elements).

The functions `read()`/`write()` take as the last argument (and they return) _the number of bytes_ - a concept originating from _size_ (not from the _length_).
If we want to use _index_ as the last argument to `read()`/`write()` then the _index_ needs to be multiplied by the size of the element (`index * sizeof(buffer[0])`)
and if we want to use the value returned by `read()`/`write()` to index the buffer then the value needs to be divided by the size of the element (`bytesRead / sizeof(buffer[0])`).
```c++
#include <unistd.h>  // Declares `read()`.

char /* or wchar_t */ buffer[6]; // The buffer to read the chars to.
int bytesRead;  // The number of bytes that have been read.
if((bytesRead = read(.., buffer,
                     sizeof(buffer) - sizeof(buffer[0]))) 
   > 0)
    // Read to the buffer up to 5 (char or wchar_t) characters
    // (up to 5 bytes for `char` or up to (>=10) bytes for `wchar_t`).
{   // The `bytesRead` contains the number of bytes actually read
    // (1..5 bytes for `char`, 1..(>=10) bytes for `wchar_t`).
    
    size_t index = bytesRead / sizeof(buffer[0]); // Calculate the index (to null-terminate).
    
    // Make sure the bytesRead is even for wchar_t case:
    // ASSERT((index * sizeof(buffer[0])) == bytesRead);
    
    buffer[index] = '\0';  // Null-terminate the sequence of
                           // (char or wchar_t) characters in the buffer.
```

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
* [g++ man page](http://man7.org/linux/man-pages/man1/gcc.1.html) (or [here](https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Dialect-Options.html#C_002b_002b-Dialect-Options) and [here](https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html)): `-Wreorder`/`-Werror=reorder`.

System Calls Failing with `EINTR`
-
_POSIX/Linux-specific_.  

A number of system calls upon failure return `-1` (or a negative value) and specify the reason of failure by setting the [`errno`](http://man7.org/linux/man-pages/man3/errno.3.html) to some value.

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
                continue; // Skip to the end of the loop body and repeat the iteration.
            case ENOMEM:
                WARN(..);
                .. // Memory allocation Failure Handling.
                return NULL; 
            case EBADF:
            case EINVAL:
                ASSERT(.."Bug in our code! errno: %d (%s)." .. errNum, strerror(errNum) ..);
                return NULL; // If NDEBUG is defined (Release Version) then the ASSERT() above is ignored
                             // and the execution reaches this line.
            default:
                ASSERT(.."Unknown or Unhandled Failure! "
                       "Bug in our code or system call API has changed. errno: %d (%s)." 
                       .. errNum, strerror(errNum) ..);
                return NULL;
        }
    }
    else if(retVal >= 0)
        break;  // Exit from the `while()` loop.
    else // (retVal <= (-2))
    {
        ASSERT(.."System call has returned an undocumented value, "
               "Bug in system call or API has changed. retVal: %d, errno: %d (%s)." 
               .. retVal, errNum, strerror(errNum) ..);
        return NULL;
    }
}   
while(1);
```
__Info:__
* [Primitives Interrupted by Signals](http://www.gnu.org/software/libc/manual/html_node/Interrupted-Primitives.html) (this link has been courteously provided by [Ivan Ponomarev](https://cppnow2019.sched.com/artist/ivantrue)).
* [`errno` Man Page](http://man7.org/linux/man-pages/man3/errno.3.html).
* [`select()` Man Page](http://man7.org/linux/man-pages/man2/select.2.html).
* [Alphabetic list of all Linux man pages](http://man7.org/linux/man-pages/dir_all_alphabetic.html).

Variable Length Arrays are C99 Feature, But Not C++
-
This item is applicable to (_strictly_) _Standard C++_ only.

Variable Length Arrays are the arrays whose length (number of elements) is a _run-time_ value (as opposed to _compile-time constant_). E.g.
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

Some more reasons are listed in [[ynovlas]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#ynovlas) _Why doesn't C++ support variable length arrays (VLAs)?_  

__Future:__  
* C++: There is an ongoing discussion about the alternatives for the Variable Length Arrays. See 
  * [[P0785R0] Runtime-sized arrays and a C++ wrapper](http://open-std.org/JTC1/SC22/WG21/docs/papers/2017/p0785r0.html),
  * [[n3810] Alternatives for Array Extensions](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3810.pdf).  
* C:  
  * [[dsa]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#dsa): _Now, keep in mind that VLAs are effectively “on the way out” of the C language. Although they have been present in the standard since C99, in C11 they have been demoted to an “optional” feature. Many compiler implementations never implemented VLAs, and many never will. C++ doesn’t support VLAs, and those compilers that do provide them do so as a non-standard extension. So, if you’re interested in writing highly portable C code, or code that will likely be migrated to C++, VLAs should be off the table, even if your compiler just happens to support them._  
  * [[ANSI_C11]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#ANSI_C11), [[C11_N1570]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#C11_N1570):  
    * 6.7.6.2 Array declarators. 4. _Variable length arrays are a conditional feature that implementations need not support_.  
    * Foreword. 6. _Major changes from the previous edition include:  
— conditional (optional) features (including some that were previously mandatory)_.  
    * 6.10.8.3 Conditional feature macros. 1. _`__STDC_NO_VLA__` The integer constant 1, intended to indicate that the implementation does not support variable length arrays or variably modified types_.  

__See Also:__
* [C99], search for "variable length".
* [[ANSI_C11]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#ANSI_C11), [[C11_N1570]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#C11_N1570), search for "variable length", "conditional feature".  
* [[dsa]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#dsa) How does dynamic stack allocation work in C, specifically regarding variable-length arrays?  
* [[ynovlas]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#ynovlas) Why doesn't C++ support variable length arrays (VLAs)?  
* [gcc](http://man7.org/linux/man-pages/man1/gcc.1.html): `-Wvla`, `-Wvla-larger-than=n`.
* [[P0785R0] Runtime-sized arrays and a C++ wrapper](http://open-std.org/JTC1/SC22/WG21/docs/papers/2017/p0785r0.html).
* [[n3810] Alternatives for Array Extensions](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3810.pdf).

Know the Danger of `alloca()`
-
The `alloca()` function allocates the space on the stack by adjusting the stack pointer and returns the pointer to the allocated block. The allocation is released upon _function return_ but _not when the pointer to such an allocation goes out of scope_. Thus if the `alloca()` is called in the loop then each loop iteration allocates more and more space on the stack which _can easily cause the stack overflow_.

Such a behavior is fundamentally differnt from the behavior of its counterparts (the ones that also allocate on the stack) -  
the ordinary arrays (whose number of elements is specified with a compile-time constant),  
and the [variable length arrays](#variable-length-arrays-are-c99-feature-but-not-c) (whose number of elements is specified with a run-time value).  
Both of these couterparts _reuse_ the space on the stack during each iteration of the loop.

The other problem with `alloca()` is that it is _non-standard_. The [variable length arrays](#variable-length-arrays-are-c99-feature-but-not-c) seem to be a better alternative since they are standard in at least [C99] and conditional (optional) in [C11].

__Demonstration in C:__
```c
#include <stdio.h>
#include <alloca.h>

void F(void)
{
  // Do the loop having the `alloca()`:
  printf("alloca():\n");
  {
    int i = 0;
    do
    {
      char a = 'a';
      void *p = alloca(0xF0);
      int b = 1;

      printf("&i == %p; &a == %p; p == %p; &b == %p\n", &i, &a, p, &b);
    }
    while(++i < 3);
  }

  // Try to reuse the stack:
  printf("alloca() repeated:\n");
  {
    int i = 0;
    do
    {
      char a = 'a';
      void *p = alloca(0xF0);
      int b = 1;

      printf("&i == %p; &a == %p; p == %p; &b == %p\n", &i, &a, p, &b);
    }
    while(++i < 3);
  }
}

int main(void)
{
  F();
  return 0;
}
```
```bash
# Save as `alloca.c`.
make alloca
# cc     alloca.c   -o alloca
./alloca 
# alloca():
# &i == 0x7ffc5c2a2790; &a == 0x7ffc5c2a278f; p == 0x7ffc5c2a2680; &b == 0x7ffc5c2a2794
# &i == 0x7ffc5c2a2790; &a == 0x7ffc5c2a278f; p == 0x7ffc5c2a2580; &b == 0x7ffc5c2a2794
# &i == 0x7ffc5c2a2790; &a == 0x7ffc5c2a278f; p == 0x7ffc5c2a2480; &b == 0x7ffc5c2a2794
# alloca() repeated:
# &i == 0x7ffc5c2a2790; &a == 0x7ffc5c2a278f; p == 0x7ffc5c2a2380; &b == 0x7ffc5c2a2794
# &i == 0x7ffc5c2a2790; &a == 0x7ffc5c2a278f; p == 0x7ffc5c2a2280; &b == 0x7ffc5c2a2794
# &i == 0x7ffc5c2a2790; &a == 0x7ffc5c2a278f; p == 0x7ffc5c2a2180; &b == 0x7ffc5c2a2794
cc --version
# cc (Ubuntu 5.4.0-6ubuntu1~16.04.10) 5.4.0 20160609
```
The demonstration shows that all the variables reuse the space on the stack (stay at the same addresses) in different iterations and in different loops, but the allocation made with `alloca()` and pointed to by the `p` pointer _changes_ (consumes more and more space on the stack).

The `alloca()` function is described in Linux man pages (e.g. [here](http://man7.org/linux/man-pages/man3/alloca.3.html), [here](https://linux.die.net/man/3/alloca)) and [MSDN page for `alloca()`](https://docs.microsoft.com/en-us/cpp/build/alloca?view=vs-2017) (which referes to [`_alloca()`](https://docs.microsoft.com/en-us/cpp/c-runtime-library/reference/alloca?view=vs-2017)).

Thanks to [Andrey Karpov](https://www.viva64.com/en/b/a/andrey-karpov/) for explaining and demonstrating this.

__More Info:__
* PVS-Studio Diagnostics [V505](https://www.viva64.com/en/w/v505/). The 'alloca' function is used inside the loop. This can quickly overflow stack.  
* [gcc](http://man7.org/linux/man-pages/man1/gcc.1.html): `-Walloca`, `-Walloca-larger-than=n` (`This option also warns when "alloca" is used in a loop`), `-fstack-protector` (`Emit extra code to check for buffer overflows, such as stack            smashing attacks`), `-mwarn-dynamicstack` (`Emit a warning if the function calls "alloca" ..`), search for `alloca`.

Know the Special Member Functions
-
* C++98/03: [[EC++3]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#EC++3) Chapter 2: Constructors, Destructors, and Assignment Operators.
* C++11: What the Move Semantics Are. Part 1 [[ms1]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#ms1), Part 2 [[ms2]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#ms2).
* C++11: [[EMC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#EMC++) Item 17: Understand special member function generation.
* [The rule of three/five/zero](https://en.cppreference.com/w/cpp/language/rule_of_three).

Know All the Effects of the Empty Destructor
-
Some people were taught to always write the destructor just in case, even if it does nothing (is empty). I aslo heard that some compilers and/or static analyzers were complaining if a class had no destructor.

Starting in C++11 the explicit destructor 
* disables the (implicit, by-compiler) generation of the move operations,
* makes the generation of the copy operations _deprecated_ (i.e. _disabled_ in some future version of C++).

See the end (_Things to Remember_ section) of [[EMC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#EMC++) Item 17: Understand special member function generation, fragments:
* Move operations are generated only for classes lacking explicitly declared move operations, copy operations, and a _destructor_.
* Generation of the copy operations in classes with an explicitly declared _destructor_ is deprecated.

__Broader Picture:__  
* [Know the Special Member Functions](#know-the-special-member-functions).
* When the empty destructor may be needed:
  * If the destructor needs to be _virtual_ or _pure virtual_. See 
    + [[EC++3]](book_list.md#EC++3) Item 7: Declare destructors virtual in polymorphic base classes;
    + [Rule of zero](https://en.cppreference.com/w/cpp/language/rule_of_three).
  * Pimpl-Idiom-specific, [[MExcC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#MExcC++):
    + _Item 30: Smart Pointer Members, Part 1: A Problem with `auto_ptr`_, section "What About `auto_ptr` Members?", last but one paragraph on paper page 185: "If you don't want to provide the definition of Y, you must write the X2 destructor explicitly, even if it's empty".
    + _Item 31: Smart Pointer Members, Part 2: Toward a `ValuePtr`_, section "A Simple `ValuePtr`: Strict Ownership Only", first regular paragraph: "Either the full definition of Y must accompany X, or the X destructor must be explicitly provided, even if it's empty".

There Should Be a Strong Reason for Writing the Destructor
-
If you use Smart Pointers and other Smart Resource Releasers (see below) then you don't need the explicit (i.e. manually written) destructor, the implicit (i.e. the compiler-generated) one will do the right thing in the [right order](https://github.com/kuzminrobin/code_review_notes/blob/master/cpp_design_bookmarks.md#know-about-the-compilers-resource-allocation-and-deallocation-order).  
See also [Rule of zero](https://en.cppreference.com/w/cpp/language/rule_of_three).

_The explicit destructor is a warning sign._

__Broader Picture:__  
* [Know the Special Member Functions](#know-the-special-member-functions).
* Smart Pointers:
  * C++98/03: [[EC++3]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#EC++3)
    * Chapter 3: Resource Management
    * Item 29: Strive for exception-safe code
    * Item 45: Use member function templates to accept "all compatible types."
    * Search for "smart" in the e-book
  * C++98/03: [[MEC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#MEC++) _Item 28: Smart pointers_ (and other items referred to in Item 28).
  * C++11/14: [[EMC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#EMC++) Chapter 4. Smart Pointers
  * [Boost.SmartPtr: The Smart Pointer Library](https://www.boost.org/doc/libs/1_68_0/libs/smart_ptr/doc/html/smart_ptr.html).
* Smart Resource Releasers: C++98/03: [[gcwywescf]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#gcwywescf).

Polymorphic Behavior of Non-Virtual Destructor
-
`2016.03.23` [[crto]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#crto) Jacek Galowicz. _Const References to Temporary Objects_.  
`2000.12.01` [[gcwywescf]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#gcwywescf) Andrei Alexandrescu and Petru Marginean. _"Generic: Change the Way You Write Exception-Safe Code — Forever"_. Search for "how to achieve polymorphic behavior of the destructor" on page 2 of 3.

Know the Peculiarities of Writing the Assignment Operator
-
When writing the _Copy-Assignment Operator_ the following logic needs to be applied.
* Know the [Special Member Functions](#know-the-special-member-functions) (in particular [[EC++3]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#EC++3) Item 5: Know what functions C++ silently writes and calls, [[EMC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#EMC++) Item 17: Understand special member function generation) and _try to avoid writing the explicit assignment operator_.
* If you have to write then _try to avoid the [Copy-And-Swap Idiom](https://stackoverflow.com/a/3279550/6362941)_ (e.g. if your Assignment Operator does not make throwing calls) since the idiom creates a copy thus lowering down the performance.
* Use the [Copy-And-Swap Idiom](https://stackoverflow.com/a/3279550/6362941) _as the last resort_.
* Don't forget to _call the parent class' Assignment Operator_, see [[EC++3]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#EC++3) Item 12: Copy all parts of an object.

TODO: Review for the _Move-Assignment Operator_.  

__Broader Picture:__  
* [Know the Special Member Functions](https://github.com/kuzminrobin/code_review_notes/blob/master/cpp_design_bookmarks.md#know-the-special-member-functions).

Inlining
-
I often see the code like this:
```c++
// Header "B.h":
class B
{
// non-private:
    void b(); // Declaration of a member function.
};    
```
```c++
// File "B.cpp":
void B::b() // Definition of the member function.
{ 
    /* Very short or empty implementation. */ 
}
```
Since the member function `B::b()` is very short (or empty) it can be more efficient to _inline_ that function instead of calling it.
However if the function is declared and defined separately as shown above then the compiler, while compiling the caller, does not know the body of the function `B::b()`. For example, we have `class D` derived from `class B` shown above.
```c++
// Header "D.h":
#include "B.h"
class D : public B
{
    void d();
};
```
```c++
// File "D.cpp":
#include "D.h"
void D::d()
{
    b(); // Calls the inherited `B::b()`.
}
```
During compilation of "D.cpp" the compiler runs the file through the preprocessor (which expands all the `#include` directives) 
and gets the translation unit that looks about like this:
```c++
class B
{
    void b();
};    
class D : public B
{
    void d();
};
void D::d()
{
    b();
}
```
When the compiler generates the code for `D::d()` the compiler does not see the body of function `B::b()` called from `D::d()` and cannot make a decision about inlining `B::b()` into the `D::d()`. That decision is either not made at all (and inlining is not done) or 
is postponed until linking.
In the latter case the linker knows the body of `B::b()` and has enough information to make a decision about inlining. And that will be the _linker_ who will be able to make the inlining (I call such an inlinig the _link-time inlining_).  
We know that inlining is not guaranteed. But if inlining is done then it is likely that the _compile-time inlining_ will be more efficient than the link-time inlining. In order to encourage the compile-time inlining we should move the definition of `B::b()` to the declaration site:
```c++
// Header "B.h":
class B
{
    // Now both declaration and definition of `B::b()`:
    void b() { /* Very short or empty implementation. */ }
};
```
and to remove the separate definion of `B::b()` from the "B.cpp":
```diff
1,4d0
< void B::b() // Definition of the member function.
< {
<     /* Very short or empty implementation. */
< }
```    

**Remember:**
The short member functions are good candidates for compile-time inlining. It is likely more efficient to place their definition together with the declaration.

**More about inlining:**  
[[EC++3]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#EC++3):
* Item 30: Understand the ins and outs of inlining.
* Item 2: Prefer consts, enums, and inlines to #defines.
* Item 5: Know what functions C++ silently writes and calls, fragment: "_All these functions will be both public and inline_".
* Item 44: Factor parameter-independent code out of templates, p. 214(paper)/235(file), Fragment: "_(Provided, of course, you refrain from declaring that function inline. If it’s inlined, each instantiation of SquareMatrix::invert will get a copy of SquareMatrixBase::invert’s code (see Item 30), and you’ll find yourself back in the land of object code replication.)_".
* Also search for `inline`, `inlining` in the e-book.

[[MEC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#MEC++):
* Item 24: Understand the costs of virtual functions, multiple inheritance, virtual base classes, and RTTI; p.118(paper)/135(file), middle, paragraph starting with: "The real runtime cost of virtual functions has to do with their interaction with inlining. For all practical purposes, virtual functions aren’t inlined.".
* TODO: Other `inline`, `inlining` occurrences in Item 26, and other places.
* Also search for `inline`, `inlining` in the e-book.

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
with the lowest value `std::numeric_limits<int8_t>::min()` (`INT8_MIN`) and we decrement such a value then we can get the _Undefined Behavior_! (Here and below, for brevity and simplicity, the integer promotion rules are ignored, i.e. it is assumed that the 8-bit value is _not_ extended to 16-bit or 32-bit value then decremented _without the overflow_ and then the least significant byte is saved back to the 8-bit integer, all in _fully defined_ way. We assume that any signed integer type (`int8_t`, `int16_t`, etc.) and/or all of those types can be implemented using `int` (or using a type larger than `int`) on some implementations)  
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
* (Expected at/after CppCon 2018) Talk/Video: JF Bastien, "[Signed integers are two's complement](http://sched.co/FnKN)".  
* C++Now 2018 Closing Keynote: Undefined Behavior and Compiler Optimizations, _John Regehr_ ([abstract](http://sched.co/ELaF), [slides](https://github.com/boostcon/cppnow_presentations_2018/blob/master/05-11-2018_friday/undefined_behavior_and_compiler_optimizations__john_regehr__cppnow__05112018.pdf), [video](https://youtu.be/AeEwxtEOgH0)).  
* C++Now 2014 Undefined Behavior in C++: What is it, and why do you care? _Marshall Clow_ ([video](https://www.youtube.com/watch?v=uHCLkb1vKaY)).  
* [gcc/g++](http://man7.org/linux/man-pages/man1/gcc.1.html) command line arguments: -fstrict-overflow, -Wstrict-overflow[=n], -Wno-overflow, -fwrapv, -fsanitize=signed-integer-overflow, -fsanitize=float-cast-overflow, -ftrapv, also search for "overflow".  
* `2018.08.20` [[woioingi](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#woioingi)] Davin McCall. _Wrap on integer overflow is not a good idea_.  

Info Sources About the Exceptions
-
Partially based on Deb Haldar's _Top 15 C++ Exception handling mistakes and how to avoid them_ [[t15c++ehm]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#heavy_check_mark-20160803-t15cehm-deb-haldar-top-15-c-exception-handling-mistakes-and-how-to-avoid-them-from-here), _Where do we go from here?_ section.

__Books and Articles at a High Level__
* `(Ongoing )` [C++ Exception FAQ](https://isocpp.org/wiki/faq/exceptions) on isocpp.org (still TODO).
* `1996.??.??` [More Effective C++ – 35 new ways to improve your programs and designs](https://www.amazon.com/More-Effective-Improve-Programs-Designs/dp/020163371X/ref=sr_1_1?ie=UTF8&qid=1470238686&sr=8-1&keywords=more+effective+c+meyers) – items 9 through 15 [[MEC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#MEC++).
* `1999.11.18` [Exceptional C++ – 47 Engineering puzzles, programming problems and solutions](https://www.amazon.com/Exceptional-Engineering-Programming-Problems-Solutions/dp/0201615622/ref=sr_1_1?ie=UTF8&qid=1470238861&sr=8-1&keywords=Exceptional+C%2B%2B) – items 8 through 19 [[ExcC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#ExcC++).
* `2000.12.01` [gcwywescf] Andrei Alexandrescu and Petru Marginean. ["Generic: Change the Way You Write Exception-Safe Code — Forever"](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#gcwywescf). Dr. Dobb's, December 01, 2000.
* `2001.12.17` [[MExcC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#MExcC++), Chapter "Exception Safety Issues and Techniques", Items 17 - 23.
* `2004.10.??` [C++ Coding Standards – 101 Rules, Guidelines and Best Practices](https://www.amazon.com/Coding-Standards-Rules-Guidelines-Practices/dp/0321113586/ref=sr_1_1?ie=UTF8&qid=1470238746&sr=8-1&keywords=C%2B%2B+Coding+standards) – items 68 through 75 [[C++CS]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#C++CS).
* `2015.??.??` [[e&su]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#e&su) MSDN. _Exceptions and Stack Unwinding in C++_.
* `2016.08.03` [[t15c++ehm]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#t15c++ehm) Deb Haldar. Top 15 C++ Exception handling mistakes and how to avoid them.

__Q&As__
* The [Copy-And-Swap Idiom](https://stackoverflow.com/a/3279550/6362941).

__Videos__
* `2018.05.09` C++Now 2018: Michael Spencer, ["How Compilers Reason About Exceptions"](https://cppnow2018.sched.com/event/EC7V/how-compilers-reason-about-exceptions?iframe=no&w=100%&sidebar=yes&bg=no) (to go online in 2018.06).
* `2018.05.09` C++Now 2018: Phil Nash, ["Option(al) Is Not a Failure"](https://cppnow2018.sched.com/event/EC7P/optional-is-not-a-failure) (to go online in 2018.06). Presentation [links](http://levelofindirection.com/refs/cpp-optional.html).

__Internals and ABIs__
* `2011.01.10` [[.eh_f]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#.eh_f) Airs – Ian Lance Taylor. _".eh_frame"_.

Certain Code Fragments Should Not Throw Exceptions
-
In general that's the code that is executed from the moment when the exception is thrown and until entering the `catch`-handler (i.e. during the _stack unwinding process_, see [[e&su]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#e&su)). Some of that code is implicitly generated by the compiler (hiddenly from the programmer).  
The programmer-written code fragments are  
* _destructors_, see  
Meyers' [[EC++3]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#EC++3), _Item 8: Prevent exceptions from leaving destructors_,  
Sutter's [[ExcC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#ExcC++), _Item8: Writing Exception-Safe Code - Part1_.  
* `operator delete`, `operator delete[]`, see  
Haldar's [[t15c++ehm]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#t15c++ehm), _Mistake # 5: Throwing exceptions in destructors or in overloaded delete or delete[] operator_.  
* _constructors of the exception classes_, see  
Haldar's [[t15c++ehm]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#t15c++ehm), _Mistake # 12: Throwing exception in an exception class constructor_.

C++ Exception Specifications
-

The Herb Sutter's article ["A Pragmatic Look at Exception Specifications"](http://www.gotw.ca/publications/mill22.htm) ([[pl@es]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#pl@es)) makes think that the exception specifications  
* instead of providing the _compile-time_ nothrow-guarantee (`throw()`) or throws-only-these-ones-guarantee (`throw(A, B)`)  
* provide the _run-time_ checks/asserts (which are _pessimization_, i.e. they lower down the performance).  

Another confirmation is in [[eses]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#eses).  

The Meyers' [[EMC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#EMC++), _Item 14: Declare functions `noexcept` if they won’t emit exceptions_,  
in the beginning makes one think that `noexcept` helps to optimize in certain cases (by means of replacing the copy operations with the move operations). But in the end still shows that there are no any _copile-time_ guarantees (a `noexcept`-function can call the functions without the exception specification, and the compilers (so far) don't complain) but there are _run-time_ checks ("_your program will be terminated if an exception tries to leave the function_").  
See also [[t15c++ehm]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#t15c++ehm), _Mistake # 9: Not realizing the implications of "noexcept" specification_.  

----
See also:  
* [[t15c++ehm]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#t15c++ehm) Deb Haldar. _Top 15 C++ Exception handling mistakes and how to avoid them_
* [[MEC++]](https://github.com/kuzminrobin/code_review_notes/blob/master/book_list.md#MEC++), _Item 14: Use exception specifications judiciously_.

C++ Exceptions: TODO.
-
* Concise high-level explanation of what the C++ exceptions are for, is it possible to live without the exceptions (with the exceptions completely disabled at compile time)? If yes then what is the cost?  
(Member vars that are arrays are inited by calling the default Ctor for each array element, if those Ctors acquire the resources then that acquisition can fail, how to return such a failure from the default Ctor?  
How and to who to return the failure from a Ctor of a global var? What happens if the global var Ctor throws?)
* In which cases is it _impossible_ to use the exceptions? Bare metal? (no OS to terminate the code that does not handle the exc)
* Some articles state that today's compilers generate such a code that the exceptions have _zero-cost in successful case_ (i.e. the exception is not thrown), but [Walter Bright](https://en.wikipedia.org/wiki/Walter_Bright) (creator of D) said (at http://nwcpp.org/ [November 2017 meeting](http://nwcpp.org/november-2017.html)) that _zero-cost exceptions are a myth_. Who is right? What particularly stands behind (what code is generated upon) `try`, `catch(T)`, `catch(...)`, `throw`, thrown-case, not-thrown-case, etc.? At C++Now2018's ["How Compilers Reason About Exceptions"](https://cppnow2018.sched.com/event/EC7V/how-compilers-reason-about-exceptions) session it has been recommended to read the ABI docs and the [[.eh_f]](https://github.com/kuzminrobin/code_review_notes/blob/master/article_list.md#.eh_f) Airs – Ian Lance Taylor. _".eh_frame"_ (still TODO).
* The Niall Douglas' talks about the exceptions, see the presentation [links](http://levelofindirection.com/refs/cpp-optional.html) recommended by Phil Nash during ["Option(al) Is Not a Failure"](https://cppnow2018.sched.com/event/EC7P/optional-is-not-a-failure) (TODO).

# Curious Fragments and Questions
## Abstract Class Constructors: `public`? `private`?
By default the constructors of an abstract class should be `protected`.  

The abstract class (i.e. the one having at least one pure virtual function) is not suitable for instantiation but for inheriting from it only. I.e. its constructors do not need to be `public`, right? Are there any cases when they _do_?  

Are there any cases (other than disabling the copy functions, etc.) when it makes sense to make such constructors `private`?

----
This page contains contributions from:
* [Jakub Wilk](https://github.com/jwilk) (@jwilk), [Andrey Karpov](https://www.viva64.com/en/b/a/andrey-karpov), [Ivan Ponomarev](https://github.com/ivanarh).  
