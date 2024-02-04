---
layout: post
title:  "Compile-time Sized source_location"
date:   2024-02-04 17:54:35 +0000
categories: c++
---

`std::source_location` is a powerful C++20 feature that lets you get rid
of those evil `__FILE__` and `__LINE__` macros. It also has the nice
property of being the same "colour" as a regular function, meaning
you aren't forced to `#define` your logging functions to take advantage
of file or line information.

However, it seems that `std::source_location` must be passed as a function
argument. This has the disadvantage that some calculations can no longer
be `constexpr`. Compare `constexpr auto file_size = std::size(__FILE__) - 1zu;`
with `auto file_size = std::strlen(loc.file_name())`.

The immediate thing you might try is to put it in a template:

```c++
template<std::source_location LOC = std::source_location::current()>
void f() {
    constexpr auto file_size = std::string_view{LOC.file_name()}.size();
}
```

However, `std::source_location` is not a structural type, and cannot be
used as a non-type template parameter.

So what gives? Ostensibly, `std::source_location` holds two numbers
(`line_` and `column_`) and two pointers to strings (`file_name_` and `function_name_`).
Why can't those be in a non-type template parameter?

Well, regarding the strings, the address of string literals are not supported. But
there are workarounds, namely using a NTTP to hold an array of chars:

```c++
template<std::size_t N>
struct constant_string {
    char data[N + 1zu];

    static constexpr constant_string<N> create(const char* source) {
        constant_string<N> result;
        for (std::size_t i = 0zu; i < N; ++i) {
            result.data[i] = source[i];
        }
        result.data[N] = 0;
        return result;
    }
};
```

And now, here comes the "trick": even though `std::source_location` isn't
a structural type, `constant_string` and `std::uint_least32_t` *are*. So,
you just call `std::source_location::current()` and extract the necessary
values individually:

```c++
template<std::uint_least32_t L, std::uint_least32_t C, constant_string FILE, constant_string FN>
struct constant_source_location {
    inline static constexpr std::uint_least32_t line = L;
    inline static constexpr std::uint_least32_t column = C;
    inline static constexpr const char(& file_name)[sizeof(FILE.data)] = FILE.data;
    inline static constexpr const char(& function_name)[sizeof(FN.data)] = FN.data;
};

template<
    std::uint_least32_t L = std::source_location::current().line(),
    std::uint_least32_t C = std::source_location::current().column(),
    std::size_t FILE_SIZE = std::string_view{std::source_location::current().file_name()}.size(),
    constant_string<FILE_SIZE> FILE = constant_string<FILE_SIZE>::create(std::source_location::current().file_name()),
    std::size_t FN_SIZE = std::string_view{std::source_location::current().function_name()}.size(),
    constant_string<FN_SIZE> FN = constant_string<FN_SIZE>::create(std::source_location::current().function_name())
>
consteval constant_source_location<L, C, FILE, FN> current_constant_source_location() {
    return {};
}
```

Example usage:

```c++
int main() {
    std::cout << current_constant_source_location().function_name << '\n';
    static_assert(sizeof(current_constant_source_location().function_name) == sizeof("int main()"));
}
```

[Try it on Compiler Explorer!](https://godbolt.org/z/zf3fn1Pz7)

However compiler support seems to be a bit buggy. For example, the above with the 
recently-released clang 18 does not give the correct source location (it gives the
source location of the `current_source_location` function, not `main`). Neither
gcc nor clang get `L` or `C` correct (they always are the line and column number of
the `current_constant_source_location` function too). And neither compiler correctly
propagate the default template arguments, e.g.:

```c++
template<constant_source_location LOC = current_constant_source_location()>
void f();
```

This may be a language defect if `std::source_location` was truly not meant to
be usable in template parameters. However, its special behaviour for
"default argument"s is not solely specified to be for function parameters,
so it should also work for template parameters.
