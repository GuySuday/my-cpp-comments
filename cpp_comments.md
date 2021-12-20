# Let's c++!
- [Let's c++!](#lets-c)
- [C++ Standard Versions](#c-standard-versions)
  - [C++11](#c11)
    - [*Dynamic Exception Specifications* were deprecated](#dynamic-exception-specifications-were-deprecated)
      - [Reasons](#reasons)
    - [*noexcept* specifier](#noexcept-specifier)
      - [Advantages](#advantages)
    - [`using` alias declaration](#using-alias-declaration)
  - [C++17](#c17)
- [Good practices](#good-practices)
  - [What's better?](#whats-better)
    - [include guards vs #pragma once](#include-guards-vs-pragma-once)
      - [include guards](#include-guards)
        - [Advantages](#advantages-1)
        - [Disadvantages](#disadvantages)
      - [#pragma once](#pragma-once)
        - [Advantages](#advantages-2)
        - [Disadvantages](#disadvantages-1)
    - [`std::map` element access: `at()` vs `[]`](#stdmap-element-access-at-vs-)
      - [When to use `[]` operator](#when-to-use--operator)
      - [When to use `at()` method](#when-to-use-at-method)
    - [`using` (alias declaration) vs `typedef`](#using-alias-declaration-vs-typedef)
      - [Basic Usage](#basic-usage)
      - [Function pointers and readability](#function-pointers-and-readability)
      - [Alias templates](#alias-templates)
# C++ Standard Versions
Here we'll examine the various interesting ISO C++ standard versions.
## C++11
### *Dynamic Exception Specifications* were deprecated
Adding `throw(<exception_list>)` to functions' signatures is deprecated.
#### Reasons
* It is impossible to specify the exceptions for template functions:
  ```cpp
  template<class T>
  void f(T t)
  {
    t.g();
  }
  ```
  `T::g` could throw any exception.
* Some functions can throw various number of exceptions, thus the list can be quite long and hard to maintain. \
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
    try {
      g();
    }
    catch (E1) {
      throw;
    }
    catch (E2) {
      throw;
    }
    catch (...) {
      std::unexcpected();
    }
  }
  ```
  Although it seems the compiler would make optimizations according to the exception specifications, it is just the opposite - It needs to be prepared for the case that your specifications are guaranteed to be the all the exceptions that could be thrown.
### *noexcept* specifier
Specifies whether a function could throw exceptions. It receives a compile-time boolean expression:
  * `noexcept(false)` - The function may throw exceptions. This is the default.
  * `noexcept`/`noexcept(true)` - The function doesn't throw an exception.
  
Meant to replace the empty exception specification `throw()`. `noexcept` doesn't call `std::unexpected()` like `throw()`, and may not unwind the stack
#### Advantages
* Optimizations
  * Compilers can assume `noexcept` functions don't need stack unwinding, saving a lot of compiler work. E.g: using a g++ compiler:
    ```cpp
    #include <iostream>

    class A
    {
        public:
          A() {std::cout << "A()" << std::endl;}
          ~A() {std::cout << "~A()" << std::endl;}
    };

    void f()
    {
        A a;
        throw 1;
    }
    void f_throw() throw()
    {
        A a;
        throw 1;
    }
    void f_noexcept() noexcept(true)
    {
        A a;
        throw 1;
    }

    int main()
    {
        try{
            // f();
            // f_throw();
            // f_noexcept();
        }
        catch(...){
            std::cout << "caught" << std::endl;
        }
    }
    ```
    Running `f`, the output is:
    ```
      A()
      ~A()
      caught
    ```
    Running `f_throw`, the output is:
    ```
      A()
      ~A()
      caught
      terminate called after throwing an instance of 'int'
    ```
    Running `f_noexcept`, output is:
    ```
      A()
      terminate called after throwing an instance of 'int'
    ```
    The compiler decided not to unwind the stack in case of `noexcept`, so the d'tor of `A` doesn't get called. We can only see that the compiler unwound the stack in case of `throw()`.
        
  * E.g: `std::vector` and move semantics - If an element is inserted to a full `std::vector`, a bigger vector needs to be created to replace the old on, so that the new vector will include all the elements from the old vector. One method of doing that is using copy semantics; It's safe, because in case of an exception thrown on an element copy, we can just destroy all the copied elements and free the newly allocated vector. The problem with this method is that it is relatively slow. Another method is to use move semantics; The problem is that in case on an exception thrown on element move, the original vector is in a "broken" state. This method is faster than the copy semantics. In conclusion, methods like `std::vector::push_back` need to check if the element's type's move c'tor is `noexcept`.
  
  * Compilers can make some optimizations with `noexcept` functions. E.g:
    ```cpp
    void f()
    {
      int x = 5;
      try {
        g();
        x = 10;
      }
      catch (...) {
        ...
      }
      h(x);
    }
    ```
    If `g` is `noexcept` - The compiler can assume that the body of `catch` is a dead code, because if `g` doesn't throw, the `catch` is not entered, and if `g` does throw, it's in violation of the `noexcept` specifier, thus `std::terminate` is called, and `catch` is not entered. Furthermore, the compiler can assume that `h` will be invoked with `x=10` for the same reasons.
* Readability - Assuming a `noexcept` function doesn't throw exceptions, it adds to the function's and program's readability.
* Richer than `throw()` - Thanks to the boolean parameter the `noexcept` specifier gets, one can use `is_nothrow_default_constructible` in order to determine if a template function (or class) can rely on a template class `noexcept`ness. 
  ```cpp
  template<class T>
  void f() noexcept(is_nothrow_default_constructible<T>::value)
  {
    T t;
  }
  ```

### `using` alias declaration
See [alias declaration vs typedef](#using-alias-declaration-vs-typedef)

## C++17
* ***Dynamic Exception Specifications* were removed**: See [Dynamic exception specifications deprecated (C++11)](#dynamic-exception-specifications-were-deprecated) for more details.


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

##### Advantages
 * Part of the C++ standard
 * Defining a unique macro for the header will ensure that the same file won't clash with other header files
 * Very easy to understand

##### Disadvantages
 * Requires to write 3 lines of code for each header file, 2 of each should agree on the same header
* Defining a unique macro requires manually isn't easy (Many IDEs generate UUID for each header file, solving this problem).


#### #pragma once
 This non-standard preprocessor directive serves the same purpose as include guards, but with some improvements.
* 
 
##### Advantages
  * Less code written
  * If done right(*), will prevent macro name clashes
  * On some compilers - Improvements in compilation speed

##### Disadvantages
 * Not part of the C++ standard
 * Isn't very easy to understand
 * The compiler needs to identify when two header files are different:
   * By full path: What about symlinks and hardlinks? What about systems that doesn't have the exact concepts of these links?
   * By content: Means that the content, or even a hash of it needs to be compared for all the files that use `#pragma once` - Lower performance.

### `std::map` element access: `at()` vs `[]`
Element access on `std::map` can be done in 2 ways:
* Using the `[]` operator - **Access or insert** the specified element. You should 
* Using the `at()` method - **Access** the specified element with **bounds checking** (throwing `std::out_of_range`).

#### When to use `[]` operator
This operator might be dangerous to use, as it inserts the specified element if it does not exist, so you should use it with caution. Another problem with this method, is that the map can't be const, as a value may be inserted to it.
Use it only if:
* You are prepared for a default initialization of your value in case the key doesn't exist
* Your map isn't const
  
#### When to use `at()` method
The best use of this method is when you know for certain that the key exists in the map. However, you can still use this method even if you are not certain, simply by catching the `std::out_of_range` exception. You can also consider several other options for the latter case:
* Using `find()` to check if the key exists (comparing it to `map::end()`) and then `at()` accordingly
* Using `count()` and comparing it to `0` and then `at()` accordingly

### `using` (alias declaration) vs `typedef`

#### Basic Usage
Let's define a new alias for int:
* Using `typedef`:
  ```cpp
  typedef int Pid;
  ```
* Using `using`:
  ```cpp
  using Pid = int;
  ```
One can argue which one is more readable and intuitive:
  * The alias-declaration definition looks like a variable assignment; E.g: `x = 10` - "Assign to `x` the value of `10`
  * The `typedef` definition is read like a variable declaration; E.g: `int x` - "Define a new variable from type `int` named `x`".

#### Function pointers and readability
A better example from a readability perspective, is function pointers:
Let's define a new function pointer:
* Using `typedef`:
  ```cpp
  typedef void (*FP)(int);
  ```
* Using `using`:
  ```cpp
  using FP = void (*)(int);
  ```
The alias-declaration has a more intuitive flow to the definition - "Define a new alias `FP` to be a function pointer with the following signature". But still, these two examples might not be a compelling reason to use alias-declaration over `typedef`.

#### Alias templates
Let's say we want to make an alias for a template:
  * Using `typedef`:
    ```cpp
    template<typename T>
    struct MyMap {
      typedef std::map<int, std::vector<T>> type;
    };
    ```
    Here `MyMap<T>::type` is an alias for `std::map<int, std::vector<T>>`.
  * Using `using`:
    ```cpp
    template<typename T>
    using MyMap = std::map<int, std::vector<T>>;
    ```
    Here `MyMap<T>` is an alias for `std::map<int, std::vector<T>>`.

As we can see, the `typedef` solution is more of a "hack" than a builtin method - We define an alias inside a template class definition in order to use `T` in that alias.

Another issue that shows the power of the alias-declaration is using your templated-alias inside another template (following the previous definitions):
  * Using `typedef`:
    ```cpp
    template<typename T>
    struct A {
      typename MyMap<T>::type map;
    };
    ```
    Why must we specify the keyword `typename` before the definition of the instance variable `map`? The compiler can't know for certain that `MyMap<T>::type` is a type, and not value; There can be one specialization of `MyMap` where `type` will be evaluated to a value and not a type. E.g:
    ```cpp
      class A { };
      template<>
      struct MyMap<A> {
        int type;
      }
    ```
    So the compiler can't know what it should be prepared for - a type or a value. That's where the programmer steps in and specify `typename` to clear the uncertainty. This happens because `MyMap<T>::type` is a [dependent type](https://en.cppreference.com/w/cpp/language/dependent_name).
  * Using `using`:
    ```cpp
    template<typename T>
    struct A {
      MyMap<T> map;
    };
    ```
    Comparing to the `typedef` definition, it is shorter and with less overhead. But why doesn't `MyMap<T>` is a dependent type on this case like it was on the `typedef` definition? Well that's because `MyMap<T>` was defined as an alias template, and thus must name a type and not a value.

