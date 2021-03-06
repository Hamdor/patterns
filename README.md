# MPark.Patterns

> Pattern Matching in __C++14__.

[![stability][badge.stability]][stability]
[![license][badge.license]][license]
[![wandbox][badge.wandbox]][wandbox]

[badge.stability]: https://img.shields.io/badge/stability-experimental-orange.svg
[badge.license]: http://img.shields.io/badge/license-boost-blue.svg
[badge.wandbox]: https://img.shields.io/badge/try%20it-on%20wandbox-green.svg

[stability]: http://github.com/badges/stability-badges
[license]: https://github.com/mpark/patterns/blob/master/LICENSE_1_0.txt
[wandbox]: https://wandbox.org/permlink/AHuPKUpF6mQudvr8

## Introduction

__MPark.Patterns__ is an experimental pattern matching library for __C++14__.

It determines whether a given value __matches__ a __pattern__ and, if it does,
__binds__ the desired portions of the value to a __handler__.

Pattern matching has been introduced to many programming languages outside of
the functional world, and this library draws inspiration from languages such as
Haskell, OCaml, Rust, Scala, and Swift.

```cpp
#include <functional>
#include <iostream>
#include <sstream>

#include <mpark/match.hpp>

int eval(const std::string& equation) {
  std::istringstream strm(equation);
  strm.exceptions(std::istringstream::failbit);

  int lhs, rhs;
  std::string op;
  strm >> lhs >> op >> rhs;

  using namespace mpark;
  return match(lhs, op, rhs)(
      pattern(arg, "plus", arg) = std::plus<>{},
      pattern(arg, "minus", arg) = std::minus<>{},
      pattern(arg, "mult", arg) = std::multiplies<>{},
      pattern(arg, "div", arg) = std::divides<>{});
}

int main() {
  std::cout << eval("101 plus 202") << '\n';  // prints "303".
  std::cout << eval("64 div 2") << '\n';  // prints "32".
}
```

## Basic Syntax

```cpp
using namespace mpark;
match(<expr>...)(
  pattern(<pattern>...) = <handler>,
  pattern(<pattern>...) = <handler>,
  // ...
);
```

## Types of Patterns

### Wildcard Pattern

A _wildcard pattern_ matches and ignores any value.

#### Requirements

None.

#### Syntax

  - `_` (underscore)

#### Examples

```cpp
int factorial(int n) {
  using namespace mpark;
  return match(n)(pattern(0) = [] { return 1; },
                  pattern(_) = [n] { return n * factorial(n - 1); });
}
```

### Bind Pattern

A _bind pattern_ passes any value matched by `<pattern>` to the handler.

#### Requirements

None.

#### Syntax

  - `arg(<pattern>)`
  - `arg` -- alias for `arg(_)`

#### Examples

```cpp
int factorial(int n) {
  using namespace mpark;
  return match(n)(pattern(0) = [] { return 1; },
                  pattern(arg) = [](auto n) { return n * factorial(n - 1); });
}
```

### Product Pattern

A _product pattern_ matches values that holds multiple values.

#### Requirements

The type `T` satisfies `Product` if given a variable `x` of type `T`,
  - If `std::tuple_size<T>` is a complete type, `x.get<I>()` is valid for all
    `I` in `[0, std::tuple_size<T>::value)`. Otherwise, `get<I>(x)` is valid
    for all `I` in `[0, std::tuple_size<T>::value)`.
  - `std::tuple_size<T>::value` is a well-formed integer constant expression.

__NOTE__: These requirements are very similar to the requirements for
          [C++17 Structured Bindings][structured-bindings].

#### Syntax

  - `prod(<pattern>...)`

#### Examples

```cpp
auto t = std::make_tuple(101, "hello", 1.1);

// C++17 Structured Bindings:
const auto& [x, y, z] = t;
// ...

// C++14 MPark.Patterns:
using namespace mpark;
match(t)(
    pattern(prod(arg, arg, arg)) = [](const auto& x, const auto& y, const auto& z) {
      // ...
    });
```

__NOTE__: The top-level is wrapped by a `tuple`, allowing us to write:

```cpp
void fizzbuzz() {
  for (int i = 1; i <= 100; ++i) {
    using namespace mpark;
    match(i % 3, i % 5)(
        pattern(0, 0) = [] { std::cout << "fizzbuzz\n"; },
        pattern(0, _) = [] { std::cout << "fizz\n"; },
        pattern(_, 0) = [] { std::cout << "buzz\n"; },
        pattern(_, _) = [i] { std::cout << i << std::endl; });
  }
}
```

### Sum Pattern

A _sum pattern_ matches values that holds one of a set of alternatives.

The `sum<T>` pattern matches if the given value holds an instance of `T`.
The `sum` pattern matches values of a sum type,

#### Requirements

The type `T` satisfies `Sum<U>` if given a variable `x` of type `T`,
  - If `mpark::variant_size<T>` is a complete type, `x.get<U>()` is valid.
    Otherwise, `get<U>(x)` is valid.
  - `mpark::variant_size<T>::value` is a well-formed integer constant expression.

The type `T` satisfies `Sum` if given a variable `x` of type `T`,
  - If `mpark::variant_size<T>` is a complete type, `visit([](auto&&) {}, x)` is valid.
  - `mpark::variant_size<T>::value` is a well-formed integer constant expression.

#### Syntax

  - `sum<U>(<pattern>)`
  - `sum(<pattern>)`

#### Examples

```cpp
using str = std::string;
mpark::variant<int, str> v = 42;

using namespace mpark;
match(v)(pattern(sum<int>(_)) = [] { std::cout << "int\n"; },
         pattern(sum<str>(_)) = [] { std::cout << "str\n"; });
// prints "int".
```

```cpp
using str = std::string;
mpark::variant<int, str> v = "hello world!";

struct {
  void operator()(int n) const { std::cout << "int: " << n << '\n'; }
  void operator()(const str& s) const { std::cout << "str: " << s << '\n'; }
} handler;

using namespace mpark;
match(v)(pattern(sum(arg)) = handler);
// prints: "str: hello world!".
```

### Optional Pattern

An _optional pattern_ matches values that can be dereferenced, and tested as a `bool`.

#### Requirements

The type `T` satisfies `Optional` if given a variable `x` of type `T`,
  - `*x` is a valid expression.
  - `x` is contextually convertible to `bool`.

#### Syntax

  - `some(<pattern>)`
  - `none`

#### Examples

```cpp
int *p = nullptr;

using namespace mpark;
match(p)(pattern(some(_)) = [] { std::cout << "some\n"; },
         pattern(none) = [] { std::cout << "none\n"; });
// prints "none".
```

```cpp
boost::optional<int> o = 42;

using namespace mpark;
match(o)(
    pattern(some(arg)) = [](auto x) { std::cout << "some(" << x << ")\n"; },
    pattern(none) = [] { std::cout << "none\n"; });
// prints "some(42)".
```

### Variadic Pattern

A _variadic pattern_ matches 0 or more values that match a given pattern.

#### Requirements

None.

#### Syntax

  - `variadic(<pattern>)`

#### Examples

```cpp
auto x = std::make_tuple(101, "hello", 1.1);

using namespace mpark;
match(x)(
    pattern(prod(variadic(arg))) = [](const auto&... xs) {
      int dummy[] = { (std::cout << xs << ' ', 0)... };
      (void)dummy;
    });
// prints: "101 hello 1.1 "
```

This could also be used to implement [C++17 `std::apply`][apply]:

```cpp
template <typename F, typename Tuple>
decltype(auto) apply(F &&f, Tuple &&t) {
  using namespace mpark;
  return match(std::forward<T>(t))(
      pattern(prod(variadic(arg))) = std::forward<F>(f));
}
```

and even [C++17 `std::visit`][visit]:

```cpp
template <typename F, typename... Vs>
decltype(auto) visit(F &&f, Vs &&... vs) {
  using namespace mpark;
  return match(std::forward<Vs>(vs)...)(
      pattern(variadic(sum(arg))) = std::forward<F>(f));
}
```

We can even get a little fancier:

```cpp
int x = 42;
auto y = std::make_tuple(101, "hello", 1.1);

using namespace mpark;
match(x, y)(
    pattern(arg, prod(variadic(arg))) = [](const auto&... xs) {
      int dummy[] = { (std::cout << xs << ' ', 0)... };
      (void)dummy;
    });
// prints: "42 101 hello 1.1 "
```

### Alternation Pattern

An _alternation pattern_ matches values that match any of the given patterns.

#### Requirements

None.

#### Syntax

  - `anyof(<pattern>...)`

#### Examples

```cpp
std::string s = "large";

using namespace mpark;
match(s)(
    pattern(anyof("big", "large", "huge")) = [] { std::cout << "big!\n"; },
    pattern(anyof("little", "small", "tiny")) = [] { std::cout << "small.\n"; },
    pattern(_) = [] { std::cout << "unknown size.\n"; });
```

## Pattern Guards

While pattern matching is used to match values against patterns and bind the
desired portions, pattern guards are used to test whether the bound values
satisfy some predicate.

#### Syntax

  - `when(<condition>);`

#### Examples

```cpp
using namespace mpark;
match(101, 202)(
    pattern(arg, arg) = [](auto&& lhs, auto&& rhs) { when(lhs > rhs); std::cout << "GT\n"; },
    pattern(arg, arg) = [](auto&& lhs, auto&& rhs) { when(lhs < rhs); std::cout << "LT\n"; },
    pattern(arg, arg) = [](auto&& lhs, auto&& rhs) { when(lhs == rhs); std::cout << "EQ\n"; });
// prints "LT".
```

## Related Work

  - [solodon4/Mach7](https://github.com/solodon4/Mach7)
  - [jbandela/simple_match](https://github.com/jbandela/simple_match/)

[apply]: http://en.cppreference.com/w/cpp/utility/apply
[visit]: http://en.cppreference.com/w/cpp/utility/variant/visit
[structured-bindings]: http://en.cppreference.com/w/cpp/language/declarations#Structured_binding_declaration
