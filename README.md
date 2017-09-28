# Function overload on `constexpr`

This document is a proposal for the C++ language to include overloading function names based on the _constexpr-ness_ of an argument.

## References

The document will multiply reference the proposal [P0595: The `constexpr` operator](http://open-std.org/JTC1/SC22/WG21/docs/papers/2017/p0595r0.html). It will also touch upon the Q&A session of the 2015 CppCon talk, [Eric Niebler: Ranges for the standard library](https://youtu.be/mFUXNMfaciE). Current metaprogramming novelty will touch upon [Boost.Hana](http://boostorg.github.io/hana/) as well.

--------------------

## Rationale

The current design of `constexpr` is flawed in the sense that it does manage to unify functions that are evaluated at run-time or compile-time with a single implementation (the function becomes _polymorhpic_ in some sense), however that implementation is restrictive in multiple regards:

- One is restricted to use language features only available in a constexpr context, even if the values are run-time, and evaluation will happen in such a context.
  - As it turns out, because of the polymorphic nature of the funciton, it may only use the intersection of features from the two contexts.
- One cannot make use of the compile-time constness of a value inside a function, meaning even if it were an integral value, it could not be subject to template metaprogramming.

### Intersection of features

Consider the following code:

```C++
constexpr unsigned int add(const unsigned int a,
                           const unsigned int b)
{
    return a + b;
}

int main()
{
    constexpr auto res = add(3, 4);
    return res;
}
```

All conforming C++14 compilers are expected to compile this program to `mov eax,7; ret;`, and [indeed they do](https://godbolt.org/g/J7UKqh).

However, what happens with the slight modification:

```C++
#include <limits>

constexpr unsigned int add(const unsigned int a,
                           const unsigned int b)
{
    return a + b;
}

int main()
{
    constexpr auto res = add(std::numeric_limits<unsigned int>::max(), 4);
    return res;
}
```

GCC and Clang wrongfully reject [this code](https://godbolt.org/g/nrKK3A), thinking it satisfies the excluding rule 8.20:2.6 ([rule #6 on cppreference](http://en.cppreference.com/w/cpp/language/constant_expression)) of core constant expressions in the latest draft:

> an expression whose evaluation leads to any form of core language undefined behavior (including signed integer overflow, division by zero, pointer arithmetic outside array bounds, etc). Whether standard library undefined behavior is detected is unspecified.

however unsigned integer overflow is not undefined behavior (even if it has little meaning in this functions context).

Language barbarism aside, imagine some sort of error a library developer would like to shield his/her users from. In a run-time world, such a rare incident would raise an exception and in the constexpr/compile-time world this is a matter of static assertions.

Consider the following:

```C++
#include <limits>
#include <stdexcept>

constexpr unsigned int add(const unsigned int a,
                           const unsigned int b)
{
    unsigned int c = a + b;

    if (c < a) throw std::overflow_error{ "Overflow in add." };

    return c;
}

int main()
{
    constexpr auto res1 = add(3, 4); // OK
    constexpr auto res2 = add(std::numeric_limits<unsigned int>::max(), 4); // ERROR
    return res1;
}
```

What happens with this code? Adding 3 and 4 work as expected, and compiler errors arise on `res2`. The milage on error messages [vary greatly](https://godbolt.org/g/8dQrwB).

- GCC argues that `error: expression '<throw-expression>' is not a constant expression`.
    - Precise, informative.
- Clang argues that `error: constexpr variable 'res2' must be initialized by a constant expression`
    - `note: subexpression not valid in a constant expression`
    - Good enough.
- MSVC argues that `error C2131: expression did not evaluate to a constant`
    - `note: failure was caused by call of undefined function or one not declared 'constexpr'`
    - `note: see usage of 'std::overflow_error::overflow_error'`
    - Gets hung up on the constexpr-ness of the exception objects CTOR, but fails to mention throwing is not allowed alltogether in a constant expression.

Okay, so polymorphism is defeated if we want to use the usual exception machinery for error signaling. Let's try to detect the error at compile-time.

Consider the following:

```C++
#include <limits>
#include <stdexcept>

constexpr unsigned int add(const unsigned int a,
                           const unsigned int b)
{
    unsigned int c = a + b;

    if (c < a) throw std::overflow_error{ "Overflow in add." };

    return c;
}

int main()
{
    constexpr auto res1 = add(3, 4); // OK
    constexpr auto res2 = add(std::numeric_limits<unsigned int>::max(), 4); // ERROR
    return res1;
}
```

All three compilers [choke](https://godbolt.org/g/tD6mcd) on the fact that `c < a` is not a constant expression, hence the static_assert won't work. _(This property of constexpr functions is the root source of the second big flaw in its design.)_

So currently, there is really no way of reporting errors from constexpr functions in the same polymorphic spirit as far as its original intent went.

#### P0595

This proposal would solve the error reporting problem in particular. If there were an operator that could detect the ambient context of the function, one could write something like this:

```C++
#include <limits>
#include <stdexcept>

constexpr unsigned int add(const unsigned int a,
                           const unsigned int b)
{
    unsigned int c = a + b;

    // In constexpr context use static_assert, otherwise throw
    if constexpr(constexpr())
        static_assert(c < a, "Overflow in add.");
    else
        if (c < a) throw std::overflow_error{ "Overflow in add." };

    return c;
}

int main()
{
    constexpr unsigned int big = std::numeric_limits<unsigned int>::max();
    constexpr auto res1 = add(big, 4); // COMPILE ERROR

    unsigned int also_big = big;
    auto res2 = add(also_big, 4); // RAISES EXCEPTION

    return res1;
}
```

Simple enough. Due to the nature of `if-constexpr`, the arms which are not relevant in the given context will be disregarded by the compiler, so they might as well contain invalid code.

### Compile-time arguments

Currently the only way to mandate a function argument to be a compile-time constant is to wrap it into a type which stores the value as a non-type template argument. Most notably this is the direction taken by the [Boost.Hana](http://boostorg.github.io/hana/) library with its [int_c](http://boostorg.github.io/hana/index.html#tutorial-integral-arithmetic) type. Having such a construct inside the STL, libraries would be given a common type to overload functions on when extra optimizations can trigger if a function argument is known to be compile-time constant. However, using types more complex than a simple `int` are cumbersome to use with such a library solution. Such complications will be made clear shortly.

### Non-type template parameters and wrapper arguments

Consider the following simple example, where a library implementer would like to take advantage of a known compile-time constant:

```C++
template <typename Base, typename Exp>
Base pown(Base a, Exp b)
{
    static_assert(std::is_integral_v<Exp>, "Exp is not of integral type.");

    if(b == 0) return (Base)1;
    if(a == 0) return (Base)0;

    Base result = a;
    for(Exp = 1; Exp < b; ++b)
        result *= a;
    
    return result;
}
```

There are two run-time checks inside for special cases. (There are nearly infinite ways to complicate this example, ranging from special cases for `b == 2` and `b == 3`, etc. all the way to hardcoding divide-and-conquer using SSE/AVX instrinsics.) The optimization we wish to make however requires one of the variables to be compile-time constants.

_NOTE 1: The kind of optimizations STL implementations employ are not implementable without compiler hooks; without knowing the actual state of constant propagation inside the compiler and triggering alternate code paths accordingly. std::pow is known to use intrinsic 'exp' operations on the arguments except for values like 2 and 3 where it does inlined multiplication._

_NOTE 2: In the Eric Niebler Q&A one might consider [1:05:25](https://youtu.be/mFUXNMfaciE?t=1h5m25s), where the implementation could use std::array instead of std::vector to be non-allocating/non-throwing, if the size of input params were known at compile-time._

```C++
template <typename Base, typename Exp>
Base pown(Base a, Exp b)
{
    static_assert(std::is_integral_v<Exp>, "Exp is not of integral type.");

    if constexpr(b == 0) return (Base)1;
    if constexpr(a == 0) return (Base)0;

    Base result = a;
    for(Exp i = 1; i < b; ++i) result *= a;
    
    return result;
}
```

Is the simplest possible solution, where we omit the run-time checks and force static dispatch of shortcircuited computations. This however does [not work](https://godbolt.org/g/yExa3q), because `a` and `b` are not constant expressions. (GCC is not hung up on this fact and compiles the code, though it seems highly non-conforming. I cannot even begin to guess what goes on inside the compiler.)

_NOTE: Neither could they be used as template arguments for forced, TMP loop unrolling and all sorts of other optimizations._

We could change the interface to `pown` to look like this:

```C++
template <int B, typename Base>
Base pown(Base a)
{
    static_assert(std::is_integral_v<Exp>, "Exp is not of integral type.");

    if constexpr(B == 0) return (Base)1;
    if (a == 0) return (Base)0;

    Base result = a;
    for(Exp i = 1; i < b; ++i) result *= a;
    
    return result;
}

auto res = pown<3>(5.6);
```

This way the check of the exponent being `0` has been statically dispatched. There are multiple problems with this solution:

- It obfuscates the interface to introduce overloads of pown, where arguments that used to come last now have to come first.
- Floating-point values may not participate in such optimizations.

The interface could still be "saved" with a wrapper type:

```C++
template <int B, typename Base>
Base pown(Base a, hana::int_<B> b)
{
    if constexpr(b == 0) return (Base)1;
    if (a == 0) return (Base)0;

    Base result = a;
    for(Exp i = 1; i < b; ++i) result *= a;
    
    return result;
}

auto res = pown(5.6, 3_c);
```

Floating point values though still cannot be taken into account.

## Proposal

Apologies for not speaking standardeese and using lamans terms.

### Overload resolution

I would like to propose allowing the overload of function names based on the _constexpr-ness_ of its arguments. This would involve extending the set of function overload resolution rules in the following manner: 

- Select functions only with matching constexpr qualifiers.
- Arguments may lose their constexpr qualifiers to obtain a matching function. _(Similarily to picking up constness.)_
    - Important note later at Benefits _NOTE 2_.
- When there are multiple ways to losing constexpr qualifiers to obtain a matching function, that is a compilation error with ambiguous function call.

### Benefits

With the aforementioned changes to overload resolution, one may introduce overloads such as:

```C++
template <typename Base, typename Exp>
Base pown(Base a, Exp b);

template <typename Base, typename Exp>
Base pown(Base a, constexpr Exp b);

template <typename Base, typename Exp>
Base pown(constexpr Base a, Exp b);

template <typename Base, typename Exp>
Base pown(constexpr Base a, constexpr Exp b);
```

Thus the following would become valid C++:

```C++
template <typename Base, typename Exp>
Base pown(constexpr Base a, constexpr Exp b)
{
    static_assert(std::is_integral_v<Exp>, "Exp is not of integral type.");

    if constexpr(b == 0) return (Base)1;
    if constexpr(a == 0) return (Base)0;

    Base result = a;
    for(Exp i = 1; i < b; ++i) result *= a;
    
    return result;
}
```

This way, one may use static dispatch and template meta-programming on compile-time constants inside the implementations where there is benefit to it. Users of an API as such are informed about potentially better implementations of a given function when provided constant expressions.

Long-time C++ users are accustomed to the possibility of overloading on constness (as well as overloading on the `this` pointer constness), and having constexpr overload, as a stronger form of const comes naturally.

_NOTE 1: while it may seem exhausting to add so many overloads to a single function, it is expected that not many functions exist with so many parameters that may be optimized for using meta-programming. pown I believe is special in this regard. Also, not all optimization cases need surface to the API (see API preservation)._

_NOTE 2: when a function is invoked with a constexpr argument, but there is no matching constexpr overload for the given argument, "losing its qualifier" does not infer that the function cannot be evaluated in a constexpr context. It solely serves as a means of finding a best match from the overload set._

#### API preservation

Making use of this feature need not necessarily result in updating APIs. As was mentioned in the referenced Ranges discussion Q&A: if ranges::transpose were to omit dynamically allocating when the size of the range is known to be compile time, it could still make do without surfacing the template param in the API definition:

```C++
namespace boost
{
    namespace detail
    {
        // Allocating
        template <typename R>
        auto transform_impl(R&& r, std::size_t s) {...}

        // Non-allocating
        template <typename R>
        auto transform_impl(R&& r, constexpr std::size_t s) {...}
    }

    template <typename Range>
    auto transform(Range&& range)
    {
        return detail::transform_impl(std::forward<Range>(range), range.size());
    }
}
```

Favoring overloads with `constexpr` qualifiers over those without one, the compiler will first try to evaluate Range::size() in a `constexpr` context. If it succeeds, it will pick up the corresponding non-allocating overload.

### Interaction with other featuers

#### `constexpr` function evaluation

Throughout the `pown` examples, I have omitted marking any return type with `constexpr` on purpose to be able to discuss it here.

One might be tempted to assume that a function

```C++
constexpr double f(constexpr double a) {...}

double b = f(5);
```

will be evaluated at compile time. However, the way `constexpr` functions were defined, they are only __allowed__ to be evaluated at compile-time. It might seem natural to assume that when a function is allowed to be evaluated at compile-time and all inputs __are__ compile-time constants, then the implementation __will__ evaluate it at compile-time. This change in behavior might confuse existing users of constexpr and therefore actual evaluation should still depend on the variable assigned to.

```C++
constexpr double f(constexpr double a) {...}

double b = f(5.); // still MAY be CT evaluated
constexpr double c = f(6.); // still MUST be CT evaluated
```

### Interaction with other proposals

#### P0595, the `constexpr` Operator

The `constexpr` operator proposal allows for a terse detection of being inside a `constexpr`context. This proposal targets a different aspect of facilitating metaprogramming, and thus the two features could live side-by-side.

If the implementors question is solely, "am I in a constexpr context" for the sake of chosing a throwing/allocating or non-throwing/non-allocating implementation, this proposal is not only neat, but also required, as otherwise (even with this proposal) it is not possible to detect the ambient context of the function call.

### Impact on legacy code

None. Currently no function definitions exist with `constexpr` qualifiers on their arguments, and as such when a `constexpr` variable is passed in, it simple loses its qualifier.

## Considered alternatives

There have been a few alternatives considered instead of constexpr overload.

### Modified P0595

I could imagine an alternative scenario, where the operator `constexpr()` could be given arguments which detect not the ambient context, but the constexpr-ness of a variable (the compilers internal state of constant propagation), however there would be no language semantics to convey this information further down the code.

```C++
// Note no constexpr on the function. Evaluation has nothing to do
// with making use of the constexpr-ness of an argument.
double f(int a)
{
    if constexpr(constexpr(a))
    {
        return f_impl<a>(); // ERROR: 'a' is not a constant expression.
                            //        Nothing guarantees on this line,
                            //        that 'a' is a constant expression.
    }
    else
    {
        return f_impl(a);
    }
}
```

### constant expression SFINAE

One might be able to construct a series of dummy templated implementations that blindly employ invocations like:

```C++
struct A {};
struct A_RT : A {};
struct A_CT : A_RT {};

double f(int a)
{
    return f_impl(a, A_CT{});
}
```

which trigger a series of function template instantiations, where the most special case uses `a` as a constant expression, and if that results in invalid code, it is discarded from the overload set and the one with regular run-time usage is picked up. Current expression SFINAE only discards overloads where the code results in a syntax error; it does not discard on syntactically valid, but semantically garbage code.

This solution however is neither friendly, neither does it scale (if even possible).