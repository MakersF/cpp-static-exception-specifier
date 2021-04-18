# C++ Static Exception Specifier

## Rationale

Exceptions in C++ make control flow unpredictable.
When calling a function, it's impossible to know which exceptions it might throw.
Event looking at its source code is not enough, as we'd have to inspect all the functions called from it, recursively until we reach the leaves.

Conceptually an exception is part of the contract of a function, and it could be considered one of it's return paths.
As such, I'd like to make this part of the function signature explicit.

C++ had [dynamic exception specifiers](https://en.cppreference.com/w/cpp/language/except_spec) in the past, where throwing an exception which was not declared in the specifier list resulted in `std::terminate` to be called. The feature has been removed from the language in the recent standards.
I believe the feature had limted usefulness because it didn't provide a way to better handle exceptions (due to lack of static checks), but made errors fatal (by calling `std::terminate` if an unexpected exception was indeed throw).

The below proposal doesn't aim to improve performance of exception handling (I don't have experience in an environment in which those were problems), but it might be that the sematic defined below allows for determinism and performance gains.

The goal of this document is to describe a way statically enforced exception specifierrs could work in c++, share some of the problems I see with this solution, get feedback from other memebers of the c++ community and potentially make this a real proposal if we believe this is a workable and worthwile improvement to the language.

## Overview

Functions are allowed to specify the types of the exceptions they throw by using the `throws` specifier in their signature.
If they opt-in into specifying the set of exceptions they can throw, the set must be comprehensive (only exceptions from that set can escape the function).
If an exception of a type which doesn't appear in the `throws` list might escalpe the function, the compiler produces a compilation error.
When `catch`ing exceptions from a function with a `throws` specifier, the `catch` doesn't perform any dynamic cast (it isn't allowed to catch a derived class when the function specifies it throws a base class).

### Limitations

Only value types can be thrown.
This is a very big limitation, but:
1. It simplifies the proposal a lot
2. If someone wants polymorphysm, they can use `std::unique_ptr<Interface>`.

I believe this limitation could be lifted.

### Backward compatibility

This is an additive change: functions not previously marked with `throws` do not change behaviour.
The ABI of calling a function with `throws` specified could be different from the ABI of calling a function which doesn't have `throws` specified.

The compiler should be able to produce the right code to call the function since it knows which function it's calling.
This means that a function with `throws` specifier can call functions with no `throws` specified, and viceversa (but this might come with a performance penalty).

## Idea

This presents the idea with concrete examples and code, rather than a strict specification.

While this uses the `throws` specifier, this has nothing to do with the old `throws` functionality. `throws` is just a placeholder, another syntax could be used.

### 1. Declaration

(1) A function can declare a comma separated set of exceptions types inside the `throws()` specifier. No cv or ref modifiers are allowed.
The `throws()` specifier is part of the function type (similar to `noexcept`).

```c++
void function() throws(exception_1, exception_2);
```

(2) The order in which the exception types is declared does not matter

```c++
void function() throws(exception_1, exception_2);
```
is equivalent to
```c++
void function() throws(exception_2, exception_1);
```

(3) The same type can be repeated several times, and it will be as if it was declared only once

```c++
void function() throws(exception_1, exception_1);
```
is equivalent to
```c++
void function() throws(exception_1);
```

**Rationale**: templates don't have to worry about duplication errors when adding new exceptions to the exception set

(4) A function cannot be overloaded with the throw specifier. `throws` is treated in the same way the return value is treated.

```c++
void function() throws(exception_1, exception_1);
void function() throws(exception_2);  // <-- Error, the function was previously declared with a different throws specifier
```

(5) Either `noexcept` or `throws` can be used

```c++
void function() throws(exception_1) noexcept; // <-- Error: only one between throws and noexcept can be used
```

(6) Empty `throws` is equivalent to `noexcept`

```c++
void function() throws();
```
is equivalent to
```c++
void function() noexcept;
```

(7) In the exception list, the `...` can be used to represent the set of every possible exceptions.

**Note**: `...` is just to express the concept. Any other syntax is also fine. The choice for `...` is because of it's use in `catch(...)`.

```c++
void function() throws(...); // <-- Might throw any exception
```

(8) `...` can be used in `throws` together with other exceptions. It is still equivalent to `throws(...)`

```c++
void function() throws(exception_1, ...);
```
is equivalent to
```c++
void function() throws(...);
```

**Rationale**: this simplify templates, which don't have to special case `...`.

(9) When computing which exceptions are thrown by a function, any function which is not marked with `throws` or `noexcept` is considered as to being marked with `throws(...)`.

### 2. Throwing and Catching Exceptions

(1) The exceptions thrown by an expression is the union of the exceptions thrown by it's subexpressions

```c++
int foo() throws(exception_1);
int bar() throws(exception_2);
int baz() throws(exception_3, exception_4);

foo() + foo() // throws(exception_1)
foo() + bar() // throws(exception_1, exception_2)
foo() + baz() // throws(exception_1, exception_3, exception_4)
```

(2) In general, a statement throws the union of the exception thrown in the substatements and sub expressions.

Some examples

```c++
// Throws(exception_1, exception_2, exception_3, exception_4)
for(int i = foo(); i < bar(); i++) {
    baz();
}

// Throws(exception_1, exception_2, exception_3, exception_4)
if(foo() == 0) {
    bar();
} else {
    foo();
}

// Throws(exception_1, exception_2)
{
  foo();
  bar();
}
```

(3) In the `try/catch` statement, `catch` can only catch by value.

```c++
try {
    foo();
} catch (const exception_1&) { // <-- Error: can only catch by value
}
```

**Note**: it would be ideal to remove this limitation. See the "Extension" part at the end of this proposal.

(4) Given the statement `try { stmt1 } catch ( type ) { stmt2 }`, the exceptions thrown is the `(exceptions_of(stmt1) - type) + exceptions_of(stmt2)` (where `-` and `+` are `set difference` and `set union`).
`...` works in special ways (remember, `...` represents the set of every possible exception):
1. `... - ... = empty set`: catching any exception results in no further exception being thrown
2. `... + ... = ...`: throwing any exception in addition to any exception results in any exception being thrown
3. `... + exception = ...`: throwing a specific exception in addition to any exception results in any exception being thrown
4. `... - exception = ...`: removing a specific exception to any exception results in any exception being thrown
5. `[set of exceptions] + ... = ...`: adding any exception to a set of specific exceptions results in any exception being thrown
6. `[set of exceptions] - ... = empty set`: removing any exception from a set of specific exceptions results in no exception being thrown

**Note**: with the rules above, it's impossible to represent "Any exception but ExceptionX".

```c++
// Throws()
try {
    foo();
} catch (exception_1) {

}

// Throws(exception_5)
try {
    foo();
} catch (exception_1) {
    throw exception_5();
}

// Throws(exception_4)
try {
    baz();
} catch(exception_3) {

}

// Throws(exception_4, exception_5)
try {
    baz();
} catch(exception_3) {
    throw exception_5();
}

// Thorws()
try {
    baz();
} catch(...) {

}

// Thorws(exception_5)
try {
    baz();
} catch(...) {
    throw exception_5();
}
```

(5) `throw` can be used in a `catch` block. It's equivalent to throwing the type the catch defined in th catch block.

```c++
try {
    foo();
} catch (exception_1) {
    throw;
}
```
is equivalent to
```c++
try {
    foo();
} catch (exception_1 e) {
    throw e;
}
```

(6) Question: How does `try/catch` work when calling both a function with the `throws` specifier and one without in its body?

### 4. Definition

(1) The definition must match the declaration

```c++
void function() throws(exception_1);

void function() throws(exception_2) { // <-- Error: the definition doesn't match the declaration
}
```

(2) In the definition, only exception types which are a subset of the declared exceptions in the `throws` specifier might escape the function.
If a thrown exception not declared in `throws` might escape the funtion, it is a compiler error.

```c++
// Correct: throws an exception which is declared
void function() throws(exception_1) {
    throw exception_1();
}

// Correct: the function doesn't have to throw
void function() throws(exception_1) {
    return
}

void function() throws(exception_1) {
    throw exception_2(); // <-- Error: exception_2 is not declared in the exception list
}
```

The check applies to any statement and expression in the function, not only to the exceptions thrown directly in the function body

```c++
void foo() throws(exception_1);

void bar() throws(exception_2) {
    foo(); // <-- Error: expression foo() throws exception_1, which is not declared in the exception list
}
```

Non declared exceptions can still be handled in the function.

```c++
void foo() throws(exception_1, exception_2);

// Correct, only exception_1 can escape the function
void bar() throws(exception_1) {
    try {
        foo();
    } catch (exception_2) {
        return;
    }
}
```

### 5. Variance

For ergonomy of use of this function, it's important to define some compatibility rules for functions which have a compatible signature, but different exceptions specifications.

Assuming `functionA` and `functionB` have compatible signatures, except for their `throws` specification, which are respectively `ExceptionSetA` and `ExceptionSetB`, if `ExceptionSetA` is covariant to `ExceptionSetB`, then `functionA` can be used in place of `functionB`.

**Note**: there is no covariance between function pointers in C++, this proposal doesn't suggest to add that. But the rules below could be followed for example in `std::function` to define whether a conversion constructor could be allowed, or to allow functions overriding virtual functions in a base class to define a different exception list.

We define below the rules when `ExceptionSetA` is covariant to `ExceptionSetB` (and is safe to use `functionA` in place of `functionB`):
1. if `ExceptionSetB` contains `...`
2. if `ExceptionSetA` doesn't contain `...` and is a subset of `ExceptionSetB`.
    This means that it's always valid if:
    1. `ExceptionSetA` is empty
    2. `ExceptionSetA` == `ExceptionSetB` (order independent comparison)

**Note**: if both `ExceptionSetA` and `ExceptionSetB` contain `...` they are covariant (follows from rule 1.).

## Support Library

Given the feature above, there are a few library features that can be provided to make use of the function.

**Note**: these utilities couldn't be implemented in the way showed here (because of how template argument packs are defined in c++). These examples are only for illustration, but it's possible to implement them.

```c++
std::exceptions_thrown<F, Args...> = /* the set of possible exceptions that F(Args...) throws */

std::remove_exceptions<ExceptionList, Exceptions...> = /* the equivalent of (ExceptionList - Exceptions) as defined in the rules above */
std::add_exceptions<ExceptionList, Exceptions...> = /* the equivalent of (ExceptionList + Exceptions) as defined in the rules above */

```

Examples

```c++
template<class F, class.. Args>
std::invoke_result_of<F, Args...> foo(F&& f, Args&&... args) throws(std::remove_exceptions<std::exceptions_thrown<F, Args...>, my_exception>) {
    try {
        return std::invoke(f, args);
    } catch(my_exception e) {
        // do something
    }
}
```

---

## Implementation

Sematically, the idea above should be implementable by transforming
```c++
RetType foo(Args...) throws(exceptions...);
```
into
```c++
std::result<RetType, std::variant<exceptions...>> foo(Args...);
```
and a set of machinery to extract the exceptions from the variant and call the appropriate `catch`.
Much better implementations can be identifieds, but this is a helpful mental model to reason on what the behavior should be.

---

## Extension

The current proposal only operates with values.
While I believe this would already be a big improvement, this is also a big limitation.
The biggest problem is that a function would not be able to define that it throws a base class, and then be free to throw various different implementations of that base class.

To enable the above use cases, the proposal should be expanded to support throwing references.

Here are a set of questions once we allow references to be thrown. I welcome any suggestion on how to solve these issues.

1. Given a function uses `throws(const T&)`, how does it work to do `throw T();`?
How does it work, given `Derived` inherits from `T`, to do `throw Derived()`?
2. Is it allowed for a function to use `throws(T, const T&)`? If so, what does `throw T()` does? What does `T ex; throw ex;` does?
3. Can base classes in catch bind to derived classes in the exception list?
```c++
struct B {};
struct D1 : B {};
struct D2 : B {};

void foo() throws(D1, D2);

// What does this statement throw? throws()?
try {
    foo();
} catch (const B&) {

}
```
4. Can we have ambiguous `catch` blocks?
5. Would it be useful to be able to do `void foo() throws(auto)` similar to how a function can have an auto deduced return type?

---

## Random Ideas

### How to allow throwing by value when a function is marked with `throws(const Type&)`? (if this is a desirable feature)

We need a place where to put the value. The value might not be of type `Type`, but it might be of a derived type, so the size is not predictable. We still want to allow this without a heap alloction.

1. The compiler emits in each object file the max of the sizes of the exceptions thrown in it, let's call it `obj_max_ex_size`.
2. The linker computes the max of all the `obj_max_ex_size`, and reserves a location in the binary with that size.
3. The compiler would create the exceptions which are thrown into that location
