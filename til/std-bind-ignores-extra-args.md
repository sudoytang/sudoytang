---
title: "std::bind Silently Drops Extra Arguments"
tags: [cpp]
---

# std::bind Silently Drops Extra Arguments

I passed the wrong member function to a callback, and it compiled:

```cpp
#include <functional>
#include <cstdint>

using namespace std::placeholders;

using FnType = std::function<void(uint64_t, uint8_t*, uint64_t, bool, bool, uint32_t)>;
void recv(FnType f) { (void)f; }

struct S {
    void act1(uint64_t, uint8_t*, uint64_t, bool, bool, uint32_t) {}
    void act2(uint64_t, uint8_t*, uint64_t) {}
    void bind() {
        recv(std::bind(&S::act2, this, _1, _2, _3));  // meant act1, still compiles
    }
};
```

`recv` wants a six-argument callable. `act2` takes three. No error, no warning
(GCC 14, `-Wall -Wextra`). When `recv` calls it, `act2` gets the first three
arguments and the last three are thrown away.

## Why: the bind object's `operator()` is a variadic template

The object returned by `std::bind` has, roughly,
`template <class... Args> R operator()(Args&&... args)`. It accepts any number
of arguments. Whether a call compiles depends only on which arguments the body
actually uses, and the standard says arguments not referenced by a placeholder
are simply ignored.

The rule is about which placeholders appear, not "the first N":

- The caller must pass at least K arguments, where K is the largest placeholder
  number used (`_3` means at least 3).
- Positions referenced by a placeholder must convert to the target's parameter
  types.
- Unreferenced positions, before or after, are ignored and never type-checked.
- Bound values such as `this` do not use up call-argument positions.

```cpp
void g(int x);
auto b = std::bind(g, _3);
b(nullptr, "any", 42);           // OK: calls g(42)
b(1, 2, 42, 3.0, "extra");       // OK: calls g(42)
b(1, 2);                         // error: no _3
```

## How it is implemented

libstdc++'s `_Bind` stores the callable and the bound arguments in a tuple.
On a call it packs the actual arguments with `std::forward_as_tuple`, then
maps every bound argument to a value:

- placeholder `_j` → `std::get<j-1>` of the argument tuple;
- nested bind expression → call it with the same arguments;
- `std::reference_wrapper` → unwrap to a reference;
- anything else → the stored value.

Then it `std::invoke`s the target. A stripped-down version (placeholders and
stored values only) that compiles:

```cpp
template <class Bound, class Tuple>
decltype(auto) select(Bound& bound, Tuple& in) {
    constexpr int j = std::is_placeholder<std::remove_cv_t<Bound>>::value;
    if constexpr (j > 0) return std::get<j - 1>(std::move(in));  // placeholder _j
    else return (bound);                                          // stored value
}

template <class F, class... BoundArgs>
struct MiniBind {
    F f_;
    std::tuple<BoundArgs...> bound_args_;

    template <class... Args>
    decltype(auto) operator()(Args&&... args) {
        auto in = std::forward_as_tuple(std::forward<Args>(args)...);
        return std::apply([&](auto&... bound) -> decltype(auto) {
            return std::invoke(f_, select(bound, in)...);
        }, bound_args_);
    }
};
```

Tuple elements that no `std::get<j-1>` picks are never used, so they are never
checked. There is no magic; the standard just doesn't ask for an arity check.

## Why it fits into a fixed-signature `std::function`

`std::function<R(A...)>`'s converting constructor is itself a template. It
only requires that the callable be invocable with `A...` and that the result
convert to `R`. The bind object passes that check.

Type erasure then generates an invoker for exactly `A...`, roughly
`R invoke(storage&, A... a) { return f(std::forward<A>(a)...); }`. That
**instantiates** (not specializes) the bind object's `operator()` once, for
those six types. After that, the bind type is erased.

So the dropping happens only at wrap time. `std::function::operator()` is a
non-template with a fixed signature, and extra arguments to it don't compile:

```cpp
std::function<void(int, int)> f = std::bind(g, _1);  // OK
f(1, 2);       // OK
f(1, 2, 3);    // error: no match for call
```

The original bind object is still a template, so it can still be called with
other arities. Only the copy inside the `std::function` is locked to one.

## Lesson: prefer a lambda

A lambda's parameter list is fixed, so the same mistake is a compile error:

```cpp
recv([this](uint64_t a, uint8_t* b, uint64_t c) { act2(a, b, c); });
// error: could not convert '<lambda ...>' to 'FnType'
```

This is one more reason to prefer lambdas over `std::bind`.
