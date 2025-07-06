# Proposal for `std::construct` Function Object

## I. Motivation

C++ users can convert member functions to funtion objects with std::mem_fn(),
partially bind arguments to functions or function objects with std::bind,
std::bind_front(), and std::bind_back(), type-erase them with std::function<>,
and use them to transform ranges with std::views::transform. But none
of that can be done directly to class constructors; a helper function
must be used to "downgrade" the constructor into a function.

The proposed `std::construct<>` is a utility function object that provides
a convenient, generic mechanism to convert a constructor overload set into
a function object, thereby allowing all the existing tooling for function
objects to be brought to bear on it.

## II. Example Problem

Imagine you have a range of size_t and you wish to return
a vector of vectors, with the sizes given from the given range.
Naive code can look like:

```c++
    std::vector<std::vector<int>> result;
    result.reserve(std::distance(input));
    for (auto sz : input) {
         result.emplace_back(sz);
    }
```

However, this is unsatisfying. The input range may be an input_range,
which does not afford two passes (one for std::distance, one for the for
loop). The emplace_back loop is less efficient than constructing the vector
from a range.

A modern range-based solution would look like

```c++
   auto result = input
       | std::views::transform([] (size_t sz) {
           return std::vector<int>(sz);
       }
       | std::ranges::to<std::vector>();
```

This is still unsatisfying, as the lambda is not concise.

## III. Proposed Solution: `std::construct`

### A. Function Signature

`std::construct<T>` evaluates to a function object that perfectly
forwards its arguments to T's constructors.

```c++
namespace std {

    template <typename T>
    inline constexpr auto construct = [] <typename... Args> (Args&&... args) -> T
    {
        return T(std::forward<Args>(args)...);
    };
}
```

### B. Semantics

- Constructs an object of type `T` using perfect forwarding
- Supports any constructor of `T`
- Returns the constructed object by value
- Works with both trivial and complex types
- `constexpr`-compatible

## IV. Example Usage

```c++
    // Basic usage
    auto str = std::construct<std::string>("Hello");

    // Complex type construction
    struct Complex {
        int x, y;
        Complex(int a, int b) : x(a), y(b) {}
    };
    auto comp = std::construct<Complex>(10, 20);

    // Composability with std::bind_front
    auto make_imag = std::bind_front(std::construct<Complex>, 0);
    auto sqrt_minus_one = make_imag(1);

    // Updated example from above
    auto result = input
        | std::views::transform(std::construct<std::vector<int>>)
        | std::ranges::to<std::vector>();

```

## V. Design Considerations

### Advantages
- Type-safe
- No memory allocation overhead
- Works with any constructible type
- Composable with the rest of the functional library

### Potential Concerns
- Might be seen as redundant with [P3312](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3312r1.pdf)

## VI. Implementation

Reference implementation:
```cpp
    template <typename T>
    inline constexpr auto construct = [] <typename... Args> (Args&&... args) -> T
    {
        return T(std::forward<Args>(args)...);
    };
```

## VII. Wording

TBD

## VIII. Complexity

- Time Complexity: O(1) construction time
- Space Complexity: No additional space overhead

## IX. Proposed Standardization

Recommend inclusion in the `<functional>` header in a future C++ standard revision.

## X. Acknowledgments

Thanks to Arthur O'Dwyer for correcting an ealier version on the mailing list, and
to Claude with assistance on this draft.