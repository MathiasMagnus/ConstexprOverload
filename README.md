# Function overload on `constexpr`

This document is a proposal for the C++ language to include overloading function names based on the _constexpr-ness_ of an argument.

--------------------

## Rationale

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
    for(Exp = 1; Exp < b; ++b) result *= a;
    
    return result;
}
```

Is the simplest possible solution, where we omit the run-time checks and force static dispatch of shortcircuited computations. This however does not work, because `a` and `b` are not _constexpr_ variables. (Neither could they be used as template arguments for forced, TMP loop unrolling.)

We could change the interface to `pown` to look like this:

```C++
template <int B, typename Base>
Base pown(Base a)
{
    static_assert(std::is_integral_v<Exp>, "Exp is not of integral type.");

    if constexpr(B == 0) return (Base)1;
    if (a == 0) return (Base)0;

    Base result = a;
    for(Exp = 1; Exp < b; ++b) result *= a;
    
    return result;
}

auto res = pown<3>(5.6);
```

This way the check of the exponent being `0` has been statically evaluated. There are multiple problems with this solution:

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
    for(Exp = 1; Exp < b; ++b) result *= a;
    
    return result;
}

auto res = pown(5.6, 3_c);
```

Floating point values though still cannot be taken into account.

## Proposal

Apologies for not speaking standardeese and using lamans terms.

### Overload resolution

I would like to propose extending the set of function overload resolution rules in the following manner: 

- Select functions only with matching constexpr qualifiers.
- Arguments may lose their constexpr qualifiers to obtain a matching function. _(Similarily to picking up constness.)_
- When there are multiple ways to losing constexpr qualifiers to obtain a matching function, that is a compilation error with ambiguous function call.

## Benefits

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

This way, one may use static dispatch and template meta-programming on compile-time constants inside the implementations where there is benefit to it.

Readers of an API are encouraged to properly mark their variables with constexpr, as they are informed about potentially better implementations of a given function when the information is provided.

Long-time C++ users are accustomed to the possibility of overloading on constness (as well as overloading on the `this` pointer constness), and having constexpr overload, as a stronger form of const comes naturally.

_NOTE: while it may seem exhausting to add so many overloads to a single function, it is expected that not many functions exist with so many parameters that may be optimized for using meta-programming. pown I believe is special in this regard. Also, not all optimization cases need surface to the API (see API preservation)._

### API preservation

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

## Interaction with other featuers

### `constexpr` function evaluation

I have omitted marking any return type with `constexpr` on purpose to be able to discuss it here.

One might be tempted to assume that a function

```C++
constexpr double f(constexpr double a) {...}

double b = f(5);
```

will be evaluated at compile time. However, the way `constexpr` functions were defined, they are only __allowed__ to be evaluated at compile-time. It might seem natural to assume that when a function is allowed to be evaluated at compile-time and all inputs __are__ compile-time constants, then the implementation __will__ evaluate it at compile-time. This change in behavior might confuse existing users of constexpr and therefore actual evaluation should still depend on the variable assigned to.

```C++
constexpr double f(constexpr double a) {...}

double b = f(5.); // still MAY be CT evaluable
constexpr double c = f(6.); // still MUST be CT evaluable
```

## Interaction with other proposals

### P0595, the `constexpr` Operator

The `constexpr` operator proposal allows for a terse detection of being inside a `constexpr`context. This proposal targets a different aspect of facilitating metaprogramming, and thus the two features could live side-by-side.

If the implementors question is solely, "am I in a constexpr context" for the sake of chosing a throwing/allocating or non-throwing/non-allocating implementation for the sake of performance, this proposal is not only neat, but also required, as otherwise (even with this proposal) it is not possible to detect the ambient context of the function call.

#### Modified P0595

I could imagine an alternative scenario, where the operator `constexpr()` could be given arguments which detect not the ambient context, but the constexpr-ness of a variable (the compilers internal state of constant propagation), however there would be no language semantics to convey this information for optimization purposes.

```C++
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
        return f(a);
    }
}
```

## Impact on legacy code

None. Currently no function definitions exist with `constexpr` qualifiers on their arguments, and as such when a `constexpr` variable is passed in, it simple loses its qualifier.