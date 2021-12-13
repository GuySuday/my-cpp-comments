# Let's c++!
- [Let's c++!](#lets-c)
- [C++ Standard Versions](#c-standard-versions)
  - [C++11](#c11)
  - [C++17](#c17)
- [Good practices](#good-practices)
  - [What's better?](#whats-better)
    - [include guards vs #pragma once](#include-guards-vs-pragma-once)
      - [include guards](#include-guards)
        - [The advantages](#the-advantages)
        - [The disadvantages](#the-disadvantages)
      - [#pragma once](#pragma-once)
        - [The advantages](#the-advantages-1)
        - [The disadvantages](#the-disadvantages-1)
# C++ Standard Versions
Here we'll examine the various interesting ISO C++ standard versions.
## C++11
* ***Dynamic Exception Specifications* were deprecated**: Adding `throw(<exception_list>)` to functions' signitures is deprecated. **Some of the reasons**:
    * It is impossible to specify the exceptions for template functions:
        ```cpp
          template<class T>
          void f(T t)
          {
            t.g();
          }
        ```
      `T::g` could throw any exception.
    * Some functions can throw various number of exceptions, thus the list can be quite long and hard to maintain.
    E.g: Removing an exception from a method of a base class' exception's specification list could affect all the derived classes'.
    * May limit compiler optimizations - Consider the following example:
        ```cpp
            void f() throw(E1,E2)
            {
              g();
            }
        ```
        Functionally the compiler will generate something like:
        ```cpp
            void f()
            {
              try
              {
                g();
              }
              catch (E1)
              {
                throw;
              }
              catch (E2)
              {
                throw;
              }
              catch (...)
              {
                std::unexcpected();
              }
            }
        ```
        Although it seems the compiler would make optimizations according to the exception specifications, it is just the opposite - It needs to be prepared for the case that your specifications are guaranteed to be the all the exceptions that could be thrown.

## C++17
* ***Dynamic Exception Specifications* were removed**: See [C++11](#c11) for more details.


# Good practices
## What's better?
### include guards vs #pragma once
During compilation some header files may be included multiple times, resulting in compilation error: *redefinition of \<symbol\>*. This is inevitable in large projects. So we need a way to insure every header file is included only once.

 #### include guards
*file.h*:
 ```cpp
#ifndef HEADER_GUARD_FOR_FILE
#define HEADER_GUARD_FOR_FILE
 ...
#endif /* HEADER_GUARD_FOR_FILE */
 ```
 The first time *file.h* is going to be included the `#ifndef HEADER_GUARD_FOR_FILE` condition is going to be true, so the macro `HEADER_GUARD_FOR_FILE` will be defined. \
 Every time after that, `HEADER_GUARD_FOR_FILE` is already defined, so `#ifndef HEADER_GUARD_FOR_FILE` condition is going to be false, thus the code inside this include guard is not going to be included.

 ##### The advantages
 * Part of the C++ standard
 * Defining a unique macro for the header will ensure that the same file won't clash with other header files
 * Very easy to understand

 ##### The disadvantages
 * Requires to write 3 lines of code for each header file, 2 of each should agree on the same header
* Defining a unique macro requires manually isn't easy (Many IDEs generate UUID for each header file, solving this problem).


 #### #pragma once
 This non-standard preprocessor directive serves the same purpose as include guards, but with some improvements.
* 
 
  ##### The advantages
  * Less code written
  * If done right(*), will prevent macro name clashes
  * On some compilers - Improvments in compilation speed

 ##### The disadvantages
 * Not part of the C++ standard
 * Isn't very easy to understand
 * The compiler needs to identify when two header files are different:
   * By full path: What about symlinks and hardlinks? What about systems that doesn't have the exact concepts of these links?
   * By content: Means that the content, or even a hash of it needs to be compared for all the files that use `#pragma once` - Lower performance.