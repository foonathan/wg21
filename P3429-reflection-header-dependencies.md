---
title: "&lt;meta&gt; should minimize standard library dependencies"
document: P4329R0
date: 2024-10-15
audience: LEWG
author:
  - name: Jonathan Müller (think-cell)
    email: <foonathan@jonathanmueller.dev>
---

# Abstract

[@P2996 - Reflection for C++26] requires library support in the form of the `<meta>` header.
As specified, this library API requires other standard library facilities such as `std::vector`, `std::string_view`, and `std::optional`.
We propose minimizing these standard library facilities to ensure more wide-spread adoption.
In our testing, the proposed changes have only minimal impact on user code.

# Background

[@P2996] adds reflection library support in form of the `<meta>` header.
The functions introduced there are essentially wrappers around compiler built-ins and thus cannot be implemented by users:
People that want to use reflection have to use `<meta>` or directly reach for the compiler built-ins.
`<meta>` thus has the same status as `<type_traits>`, `<coroutine>`, `<initializer_list>`, and `<source_location>`.
Yet, unlike those headers, which carefully avoided standard library dependencies, `<meta>` does not.

Right now, it is specified to include:

* `<initializer_list>`
* `<ranges>`
* `<string_view>`
* `<vector>`

In addition, the interface requires:

* `std::size_t` in various places
* `<compare>` (for `member_offsets` defaulted comparison operator)
* `std::optional` (in `data_member_options_t`)
* `std::span` (in `define_static_array`)

This means every implementation of `<meta>` is required to include the specified headers, and provide the definitions for the specified types.
In addition, the compiler needs to be able to construct and manipulate objects of various standard library types at compile-time.

# Motivation

Reflection is primarily a language feature that requires some standard library APIs.
However, the C++ language has more users than the C++ standard library; developers in some domains (e.g. gamedev) make heavy use of C++ but avoid using the standard library.
It is not for us in the committee to say whether they are justified in their reasons, but we have a duty to represent the interests of all our users and not just the subset of users that have representation in WG21.
If we can better support everyone at minimal cost, we should do so or risk seeming even more out of touch.

To that end, the `<meta>` header should minimize standard library dependencies.
This has the following advantages:

* **Better compile-times due to reduced header inclusion:** A translation unit that does not require `<ranges>` or `<vector>` should not be forced to pay the (significant!) price for including them.
  This is especially important as reflection by design has to be used in a header file, and we anticipate wide-spread use of reflection in core utility libraries included in most translation units.
  If this comes with the entirety of `<ranges>`, this can cause significant increases in compile-time for projects that have not adopted it.

  For example, [@lexy] is a C++ parser combinator library which avoids using standard library headers in the core library code as the metaprogramming alone makes compile-times slow enough.
  Right now, a clean rebuild takes 61.979 s ±  1.646 s.
  Including `<ranges>`, `<vector>` and `<string_view>` in a core header file, where reflection might be used to replace `__PRETTY_FUNCTION__` hacks, increases it to 86.786 s ±  0.468 s.
  Actually starting to use reflection features will slow it down further.
  This is makes reflection unusable.

  Modules may solve this problem in the long-term, but adopting modules requires changes to the build system, which makes it more difficult than "just" updating the compiler to start using reflection.
  It is not entirely unreasonable to think that some companies will start using reflection before they start using modules.

* **Better compile-times due to simpler API types:** All reflection APIs are `consteval`, so if a function e.g. returns a `std::vector`, the compiler has to construct this object at compile-time.
  This is expensive, as the memory layout and behavior of `std::vector` is controlled by standard library implementations the compiler frontend does not necessarily have full control over.
  If the result type were a custom type in control of the compiler instead, compiler implementers can pick an efficient representation that is easier to work with at compile-time.

  As a concrete example, `std::meta::members_of()` returns a `std::vector<std::meta::info>`.
  In the [@bloomberg-clang] implementation, the underlying compiler API essentially exposes a `begin()` + `next()` function that is wrapped into a range type and passed to `std::vector`'s constructor.
  The compile-time interpreter then needs to allocate compile-time memory and simulate iterator machinery to construct a `std::vector` object.
  We propose that the API instead returns a `std::meta::info_array` whose layout is completely in control by the compiler.
  Then the compiler can allocate an array of `std::meta::info` objects using its own implementation, and not C++ code in an interpreter, and simply expose a pointer to that array.

* **You do not pay for what you do not use:** Most use cases of e.g. `std::meta::members_of()` just call it and iterate over it.
  Yet they have to pay for a full copy of the member list living in a `std::vector` that is immediately destroyed afterwards.
  This copy is necessary in the general case as querying properties of an entity at different points in a translation unit can result in different results (e.g. due to incomplete types that become complete).
  But if you immediately iterate over it, the copy is an unnecessary cost.

  With a custom result type, a smart compiler implementation can employ copy-on-write techniques to avoid unnecessary copies.

* **Precedence:** `std::source_location::file_name()` does not return a `std::string_view`, but a `const char*` instead.
  Proposed `std::contracts::contract_violation::comment()` does not return a `std::string_view`, but a `const char*` instead.
  So why should `std::meta::identifier_of()` return a `std::string_view`?

* **Minimal downsides:** The proposed changes have minimal negative impact on user code, so the cost of applying them is not high.

We therefore propose that `<meta>` should minimize standard library dependencies.
In this paper, we exhaustively enumerate every dependency it has and suggest an alternative.
Note that we are not proposing to force the implementation to avoid standard library dependencies in their implementations.
We are just making it possible for them to do so.

All wording in this paper is relative to [@P2996].

# Implementation experience

For the purposes of testing the impact on user code, some of the proposed changes have been implemented by modifying the `<meta>` header of [@bloomberg-clang].
These changes are:

* Poll 2: Replacing `std::vector` by `std::meta::info_array`. The implementation still uses `std::vector` internally, but the interface impact becomes visible.
* Poll 4.2: Replacing `std::[u8]string_view` in return positions by `const char[8_t]*`.

Poll 1.1 has obvious user impact and does not need to be investigated; polls 1.2, 1.3, and 4.1 are non-breaking changes that widen the interface contract; poll 3 is potentially a breaking change but requires compiler support to implement; poll 5 is an obvious breaking change that provides equivalent convenience and does not need to be investigated.

The modified `<meta>` header is available here: [@prototype-impl]

We investigated the following examples from [@P2996] by replacing the `#include <experimental/meta>` with an `#include` of the above implementation:

* [enum to string](https://godbolt.org/z/TjaYrvz67) (no change needed)
* [parsing command-line options](https://godbolt.org/z/xexYdhYYa) (no change needed)
* [a simpler tuple type](https://godbolt.org/z/n85PxW7a9) (no change needed)
* [a simpler variant type](https://godbolt.org/z/Tn664Gf5d) (no change needed)
* [struct to struct of arrays](https://godbolt.org/z/58T3fhd35) (replace hard-coded type by `auto`)
* [parsing command-line options II](https://godbolt.org/z/994ood5s4) (no change needed)
* [a universal formatter](https://godbolt.org/z/P3jTv3jo9) (no change needed)
* [converting a struct to tuple](https://godbolt.org/z/bMs3P1M3z) (no change needed)
* [implementing tuple_cat](https://godbolt.org/z/qzKGrjjdY) (note: required adjustment unrelated to this paper)
* [named tuple](https://godbolt.org/z/n38TE7nTj) (no change needed)

Similarly, we investigated the following examples of [@daveed-keynote]:

* [tuple implementation](https://godbolt.org/z/76srjM8dn) (no change needed)
* [command-line argument parsing](https://godbolt.org/z/YrEf4TM1x) (additional `std::ranges::to<std::vector>` needed)

We also manually investigated the reflection implementation of [@daw_json_link], which requires an additional `std::ranges::to<std::vector>` in the implementation of a reflection function that has since been added to [@P2996].

# Baseline: Depending on `<initializer_list>`, `<compare>` and `std::size_t` is perfectly fine

Those facilities are other compiler interface headers that are perfectly fine to use.
We do not propose removing those dependencies.

# Poll 0: Remove the list of includes from the wording

The wording requires an include of `<ranges>` but the interface only requires a couple of concepts.
Yet, because it is specified in the wording, every implementation, even ones that partition their headers to avoid unnecessary dependencies like libc++, have to include `<ranges>`, and not just the smaller internal header that defines the necessary concepts.

Removing the requirement gives implementations more freedoms at the cost of users having to include the headers themselves when they need those features.
But it is likely that users which heavily use e.g. `<ranges>` will have it included already anyway.
However, users that don't use `<ranges>` will not have to pay the extra cost of the include.
As noted in the motivation, these includes alone caused a 40% increase in compile-time for [@lexy].

Requiring the include of `<initializer_list>` is not a big problem and is common for other headers.

## User impact

Users that included `<meta>` now also need to ensure they have `<ranges>`, `<vector>`, or `<string_view>` included if they want to use declarations from those headers that aren't already used in `<meta>`'s interface.

## Implementation impact

None. An implementation can still include whatever headers they want.

## Wording

Modify the new subsection in 21 [meta] after 21.3 [type.traits]:

**Header `<meta>` synopsis**

```diff
#include <initializer_list>
-#include <ranges>
-#include <string_view>
-#include <vector>

namespace std::meta {
…
```

# Range concepts

## Poll 1.1: Make the `reflection_range` concept exposition-only

The wording defines a concept `reflection_range` that is used to constrain member functions.
It is modeled by input ranges whose value and reference types are `meta::info` (references).

It is unlikely that users want to refine this concept further in a way that requires subsumption, so it is not necessary to expose it as a concept.
So at best exposing the concept is convenient for users writing generic code which also requires input ranges whose value and reference types are `meta::info` (references).
However, this problem is better solved by adding a generic `ranges::input_range_of<T>` concept, as it is a problem that is not specific to reflection.

Exposing the ad-hoc concept right now as-is would also freeze it in-place, so even if we had a `ranges::input_range_of<T>` concept, `reflection_range` would not subsume it.
Leaving it exposition-only gives us more leeway.

## User impact

Users that want to constrain a function on a range of `std::meta::info` objects need to write a similar concept themselves.

## Implementation impact

None. An implementation presumably will still add the concept, just under a different name.

### Wording

Modify the new subsection in 21 [meta] after 21.3 [type.traits]:

**Header `<meta>` synopsis**

```diff
…
  // [meta.reflection.substitute], reflection substitution
  template <class R>
-    concept reflection_range = see below;
+    concept @_reflection-range_@ = see below; // @_exposition-only_@

-  template <@_reflection-range_@ R = initializer_list<info>>
+  template <@_reflection-range_@ R = initializer_list<info>>
    consteval bool can_substitute(info templ, R&& arguments);
-  template <@_reflection-range_@ R = initializer_list<info>>
+  template <@_reflection-range_@ R = initializer_list<info>>
    consteval info substitute(info templ, R&& arguments);
…
```

And likewise replace all others of `reflection_range` with `@_reflection-range_@`.

## Poll 1.2: Change the `@_reflection_range_@` concept to match language semantics

As specified, `@_reflection_range_@` is a refinement of `ranges::input_range`, which imposes additional requirements on the iterator type,
like the existence of a `value_type` and `difference_type` or `std::iterator_traits` specialization.
Those requirements are not necessary for the range-based for-loop.

People that don't care about the standard library range concepts, still want to use reflection.
They thus might have range types that don't model any of the standard library range concepts, but are still supported by the range-based for-loop.
For an interface that is supposed to be low-level and close to the compiler, it can make sense to instead follow the language semantics, and not the standard library semantics.

## User impact

Positive, functions accept strictly more types than before.

::: cmptable

### Before

```cpp
class MyRange
{
public:
    class iterator
    {
    public:
        using value_type     = std::meta::info;
        using difference_type = std::ptrdiff_t;

        iterator();

        std::meta::info operator*() const;
        iterator& operator++();
        void operator++(int);

        bool operator==(iterator, iterator) const;
    };

    iterator begin();
    iterator end();
};

auto result = substitute(info, MyRange{});
```

### After

```cpp
class MyRange
{
public:
    class iterator
    {
    public:





        std::meta::info operator*() const;
        iterator& operator++();


        bool operator==(iterator, iterator) const;
    };

    iterator begin();
    iterator end();
};

auto result = substitute(info, MyRange{});
```

:::

## Implementation impact

Implementations need to write a custom concept instead of using `ranges::input_range`.

### Wording

Modify [meta.reflection.substitute] Reflection substitution

```diff
template <class R>
concept @_reflection_range_@ =
-  ranges::input_range<R> &&
-  same_as<ranges::range_value_t<R>, info> &&
-  same_as<remove_cvref_t<ranges::range_reference_t<R>>, info>;
+  @_see-below_@; // @_exposition-only_@
```

::: add

A type `R` models the exposition-only concept `@_reflection-range_@` if a range-based for loop statement [stmt.ranged] `for (std::same_as<info> auto _ : r) {}` is well-formed for an expression of type `R`.

[*Note*: This requires checking whether the `begin-expr` and `end-expr` as defined in [stmt.ranged] are well-formed and that the resulting types support `!=`, `++`, and `*` as needed in the loop transformation. — *end note*]

:::

## Poll 1.3: Replace `ranges::input_range`

Similarly, the `define_static_array` function is constrained by `ranges::input_range`.
With the same logic as above, this should be generalized to require only the range-based for loop to work.

## User impact

Positive, functions accept strictly more types than before (see above).

## Implementation impact

Implementations need to write a custom concept instead of using `ranges::input_range`.

### Wording

Modify [meta.reflection.define_static] Static array generation

```diff
-template<ranges::input_range R>
+template<typename R>
-    consteval span<const ranges::range_value_t<R>> define_static_array(R&& r);
+    consteval @_see-below_@ define_static_array(R&& r);
```

And change the wording as follows:

::: add

[4]{.pnum} *Constraints:* A range-based for loop statement [stmt.ranged] `for (auto&& x : r) {}` is well-formed for an expression of type `R`.

[*Note*: This requires checking whether the `begin-expr` and `end-expr` as defined in [stmt.ranged] are well-formed and that the resulting types support `!=`, `++`, and `*` as needed in the loop transformation. — *end note*]

[5]{.pnum} Given `for (auto&& x : r) {}`, let `@_D_@` be the number of times the range-based for loop is iterated, `@_T_@` be the type `decay_t<decltype(x)>`, and `@_S_@` be a constexpr variable of array type with static storage duration, whose elements are of type `const @_T_@`, for which there exists some `k ≥ 0` such that `@_S_@[k + i] @_==_@ @_ri_@` for all 0 ≤ i < D where `ri` is the value of `x` on the `i`th iteration.

[6]{.pnum} Returns: `span<const @_T_@>(addressof(@_S_@[k]), @_D_@)`

[7]{.pnum} Implementations are encouraged to return the same object whenever the same the function is called with the same argument.

:::

And update the synopsis accordingly.

# Poll 2: Replace `std::vector` by new `std::meta::info_array`

Functions that return a range of `meta::info` objects do it by returning a `std::vector<std::meta::info>` (for implementation reasons, it has to be an owning container and cannot be something like a `std::span`).
For the reasons discussed above, this is not ideal.

Instead, all functions that return a `std::vector<std::meta::info>` should instead return a new type `std::meta::info_array`.
This is a type that can only be constructed by the implementation and has whatever internal layout is most appropriate for the implementation.
Unlike `std::vector`, the proposed `std::meta::info_array` is not mutable and cannot grow in size.
This simplifies the implementation further.

For the vast majority of calls, this change is not noticeable:
All they do is iterate over the result, compose it with other views, or apply range algorithms.
For those users that do rely on having a mutable, growable container, they need to call `std::ranges::to<std::vector>`.

The proposed wording adds all members of `std::array`, except for `fill` (which doesn't make sense) and `reverse_iterator` (would introduce more standard library dependencies).
That way it also provides all functions provided by `std::ranges::view_interface` for a contiguous range (except for `operator bool`, which is weird for a container).
Alternatively, the minimum interface could model `std::initializer_list` and only provide `begin`/`end` and `size`.

## User impact

Users that want to append elements to a range of input objects or remove them from a range of input objects need to either use `views::concat`/`views::filter` or `std::ranges::to`.

In our testing, this only affected the command-line argument parsing example of Daveed's keynote [@daveed-keynote]:

::: cmptable

### Before

```cpp
static consteval auto clap_annotations_of(info dm) {
    auto notes = annotations_of(dm);
    std::erase_if(notes, [](info  ann) {
        return parent_of(type_of(ann)) != ^^clap;
    });
    return notes;
}
```

### After (option 1)

```cpp
static consteval auto clap_annotations_of(info dm) {
    auto notes = annotations_of(dm) | std::ranges::to<std::vector>();
    std::erase_if(notes, [](info  ann) {
        return parent_of(type_of(ann)) != ^^clap;
    });
    return notes;
}
```

## After (option 2)

```cpp
static consteval auto clap_annotations_of(info dm) {
    return annotations_of(dm)
        | std::views::filter([](info  ann) {
            return parent_of(type_of(ann)) == ^^clap;
        });
}
```

:::

A similar change would be needed for `daw_json_link`. However, [their use case](https://github.com/beached/daw_json_link/blob/85f5f3f3d15a27fa000733f758b157d2267a74c8/include/daw/json/daw_json_reflection.h#L24-L32) is served by `std::meta::get_public_nonstatic_data_members()`.

For completeness, we also needed to update the implementation of e.g. `std::meta::subobjects_of` which called `.append_range()` internally:

::: cmptable

### Before

```cpp
consteval auto subobjects_of(info r) -> vector<info> {
    if (is_namespace(r))
        throw "Namespaces cannot have subobjects";

    auto subobjects = bases_of(r);
    subobjects.append_range(nonstatic_data_members_of(r));
    return subobjects;
}
```

## After

```cpp
consteval auto subobjects_of(info r) -> info_array {
    if (is_namespace(r))
        throw "Namespaces cannot have subobjects";

    auto subobjects = bases_of(r) | ranges::to<vector>();
    subobjects.append_range(nonstatic_data_members_of(r));
    return subobjects;
}
```

:::

An implementation that does not use `std::vector` internally would not need to change anything.

Users that do not wish to modify the container and only iterate over it are not affected.

## Implementation impact

An implementation that does not care about decoupling dependencies just need to provide a wrapper class over `std::vector`.
In our prototype implementation, it was done in [30 lines of code](https://gist.github.com/foonathan/457bd0073cfde568e446eb4d42ec87fe#file-meta-L431-L462).
A production-ready implementation can then also optimize `std::ranges::to` to avoid additional memory allocations and copies when the user wants a `std::vector`.

## Wording

Modify the new subsection in 21 [meta] after 21.3 [type.traits]:

**Header `<meta>` synopsis**

```diff
…
namespace std::meta {
    using info = decltype(^::);

+    // [meta.reflection.info_array], info array
+    class info_array;

    …

-    consteval vector<info> template_arguments_of(info r);
+    consteval info_array template_arguments_of(info r);

    // [meta.reflection.member.queries], reflection member queries
-    consteval vector<info> members_of(info r);
+    consteval info_array members_of(info r);
-    consteval vector<info> bases_of(info type);
+    consteval info_array bases_of(info type);
-    consteval vector<info> static_data_members_of(info type);
+    consteval info_array static_data_members_of(info type);
-    consteval vector<info> nonstatic_data_members_of(info type);
+    consteval info_array nonstatic_data_members_of(info type);
-    consteval vector<info> enumerators_of(info type_enum);
+    consteval info_array enumerators_of(info type_enum);

-    consteval vector<info> get_public_members(info type);
+    consteval info_array get_public_members(info type);
-    consteval vector<info> get_public_static_data_members(info type);
+    consteval info_array get_public_static_data_members(info type);
-    consteval vector<info> get_public_nonstatic_data_members(info type);
+    consteval info_array get_public_nonstatic_data_members(info type);
-    consteval vector<info> get_public_bases(info type);
+    consteval info_array get_public_bases(info type);
}
```

Add a new section **[meta.reflection.info_array] Info array**:

::: add

```cpp
class info_array {
public:
    using value_type             = info;
    using pointer                = const info*;
    using const_pointer          = pointer;
    using reference              = const info&;
    using const_reference        = reference;
    using size_type              = size_t;
    using difference_type        = ptrdiff_t;
    using iterator = pointer;

    info_array() = delete;
    ~info_array();
    constexpr info_array(const info_array&);
    constexpr info_array(info_array&&) noexcept;
    constexpr info_array& operator=(const info_array&);
    constexpr info_array& operator=(info_array&&) noexcept;

    constexpr void swap(info_array&) noexcept;

    constexpr iterator begin() const noexcept;
    constexpr iterator end() const noexcept;

    constexpr iterator cbegin() const noexcept { return begin(); }
    constexpr iterator cend() const noexcept { return end(); }

    constexpr bool empty() const noexcept { return begin() == end(); }
    constexpr size_type size() const noexcept { return end() - begin(); }
    constexpr reference operator[](size_type i) const noexcept { return begin()[i]; }
    constexpr reference front() const noexcept { return begin()[0]; }
    constexpr reference back() const noexcept { return end()[-1]; }
    constexpr pointer data() const noexcept { return begin(); }
};
```

[1]{.pnum} The type `info_array` is a non-mutable, non-resizable container of `info` objects.

etc. etc.

:::

Update the other sections accordingly to use `info_array` instead of `vector<info>`.

# Poll 3: Replace `std::span` by `std::initializer_list`

`define_static_array` returns a `std::span` to the static array that is being defined.
To reduce dependencies, this could be `std::initializer_list`.
This has the additional benefit that you can pass it to `std::initializer_list` constructors.

It is not clear to us how `define_static_array` would be used in the first place, since you cannot actually initialize an array with it.
We could imagine changing the language to allow initilization of an array from a `consteval` `std::initializer_list` object, but allowing initialization from a `std::span` (and presumably arbitrary ranges) seems more involved.

## User impact

Users that want to e.g. index into the result of `define_static_array` need to use `.begin()[i]` instead or write `std::span(define_static_array(…))`.
Users can now use `define_static_array` to more easily initialize containers.
User that merely iterate over the result are not affected.

## Implementation impact

The implementation needs to add a way to construct a `std::initializer_list` from a pointer plus size, or change the compiler API to return `std::initializer_list` directly.

## Wording

Modify [meta.reflection.define_static] Static array generation and update the synopsis accordingly

```diff
-template<ranges::input_range R>
+template<typename R>
-    consteval span<const ranges::range_value_t<R>> define_static_array(R&& r);
+    consteval initializer_list<ranges::range_value_t<R>> define_static_array(R&& r);
```

[4]{.pnum} *Constraints*: `is_constructible_v<ranges::range_value_t<R>, ranges::range_reference_t<R>>` is `true`.

[5]{.pnum} Let `@_D_@` be `ranges::distance(r)` and `@_S_@` be a constexpr variable of array type with static storage duration, whose elements are of type `const ranges::range_value_t<R>`, for which there exists some `k ≥ 0` such that `@_S_@[k + i] == r[i]` for all `0 ≤ i < @_D_@`.

[6]{.pnum} *Returns*: [`span(addressof(S[k]), D)`]{.rm}[An `initializer_list` object `il` where `il.begin() == @_S_@ + k` and `il.end() == @_S_@ + k + D`]{.add}.

[7]{.pnum} Implementations are encouraged to return the same object whenever the same the function is called with the same argument.

# Replace `std::[u8]string_view`

## Poll 4.1 Replace `std::[u8]string_view` in argument positions by a generic argument

`define_static_string` takes a `std::[u8]string_view`. The dependency can be avoided and the function be made more general if it instead accepts any range of characters.

### User impact

Positive, functions accept strictly more types than before.

### Implementation impact

Minimal. An implementation that does not care about minimizing dependencies can just construct a `std::string` with `std::ranges::to` and pass a `std::string_view` of that to the compiler API that has already been shown to be implementable.

### Wording (if poll 1.3 has no consensus)

Update [meta.reflection.define_static] Static array generation as follows.

```diff
-consteval const char* define_static_string(string_view str);
+template<ranges::input_range R> requires same_as<ranges::range_value_t<R>, char> && same_as<remove_cvref_t<ranges::range_reference_t<R>>, char>
+consteval const char* define_static_string(R&& r);
-consteval const char8_t* define_static_string(u8string_view str);
+template<ranges::input_range R> requires same_as<ranges::range_value_t<R>, char8_t> && same_as<remove_cvref_t<ranges::range_reference_t<R>>, char8_t>
+consteval const char8_t* define_static_string(R&& r);
```

[1]{.pnum} [Let `@_str_@` be `ranges::to<string>(forward<R>(r))`.]{.add} Let `@_S_@` be a constexpr variable of array type with static storage duration, whose elements are of type `const char` or `const char8_t` respectively, for which there exists some `k ≥ 0` such that:

* [1.1]{.pnum} `S[k + i] == str[i]` for all `0 ≤ i < str.size()`, and
* [1.2]{.pnum} `S[k + str.size()]` == '\0'.

[2]{.pnum} *Returns*: `&@_S_@[k]`

[3]{.pnum} Implementations are encouraged to return the same object whenever the same variant of these functions is called with the same argument.

And update the synopsis accordingly.

### Wording (if poll 1.3 has consensus)

Update [meta.reflection.define_static] Static array generation as follows.

```diff
-consteval const char* define_static_string(string_view str);
-consteval const char8_t* define_static_string(string_view str);
+template<typename R>
+consteval const @_see-below_@* define_static_string(R&& str);
```

::: add

[1]{.pnum} *Constraints:* A range-based for loop statement [stmt.ranged] `for (std::same_as<char> auto x : r) {}` or `for (std::same_as<char8_t> auto x : r) {}` is well-formed for an expression of type `R`.

[*Note*: This requires checking whether the `begin-expr` and `end-expr` as defined in [stmt.ranged] are well-formed and that the resulting types support `!=`, `++`, and `*` as needed in the loop transformation. — *end note*]

[2]{.pnum} Given `for (auto&& x : r) {}`, let `@_D_@` be the number of times the range-based for loop is iterated, `@_T_@` be the type `decay_t<decltype(x)>`, and `@_S_@` be a constexpr variable of array type with static storage duration, whose elements are of type `const @_T_@`, for which there exists some `k ≥ 0` such that `@_S_@[k + i] @_==_@ @_ri_@` for all 0 ≤ i < D, where `ri` is the value of `x` on the `i`th iteration, and `@_S_[k + D] == '\0'`.

[3]{.pnum} *Returns:* `@_S_@[k]`.

[4]{.pnum} Implementations are encouraged to return the same object whenever the same the function is called with the same argument.

:::

And update the synopsis accordingly.

## Poll 4.2: Replace `std::[u8]string_view` as return type with `const char[8_t]*`

`[u8]identifier_of` and `[u8]display_string_of` return a `std::[u8]string_view`.
In addition, to the dependency problems, it is not guaranteed to be a null-terminated string, and even if it were, getting a null-terminated string out of a `std::string_view` requires an awkward `.data()` call and a comment explaining why it is null-terminated.
Both problems are solved by returning a `const char[8_t]*` instead, just like `std::source_location::file()` does, which is a very similar function.

The downside is that users who want one of the gazillion member functions have to manually create a `std::string_view` first.
However, most calls probably forward the resulting identifier unchanged and are unaffected.

### User impact

Users that require a `std::[u8]string_view` as opposed to a `const char[8_t]*` need to construct it manually.
In our testing we have not found any.

::: cmptable

### Before

```cpp
auto name = identifier_of(info);
… name.find(…) …
```

### After

```cpp
std::string_view name = identifier_of(info);
… name.find(…) …
```

:::

Users that require a null-terminated string get one directly.

### Implementation impact

None, the compiler API on clang for example already returns a `const char[8_t]*`.

### Wording

Modify the new subsection in 21 [meta] after 21.3 [type.traits]:

**Header `<meta>` synopsis**

```diff
…
    // [meta.reflection.names], reflection names and locations
    consteval bool has_identifier(info r);

-    consteval string_view identifier_of(info r);
+    consteval const char* identifier_of(info r);
-    consteval u8string_view u8identifier_of(info r);
+    consteval const char8_t* u8identifier_of(info r);

-    consteval string_view display_string_of(info r);
+    consteval const char* display_string_of(info r);
-    consteval u8string_view u8display_string_of(info r);
+    consteval const char8_t* u8display_string_of(info r);

    consteval source_location source_location_of(info r);
…
```

And update [meta.reflection.names] accordingly.

# Poll 5: Re-design `data_member_spec`

`data_member_spec` defines a new data member.
It only takes one mandatory attribute, the type, and optional options in an object of type `data_member_options_t`.
This aggregate type has multiple standard library dependencies:

1. The members `name`, `alignment` and `width` are `std::optional`.
2. The nested type `name_type` is specified to be constructible from anything a `std::string` or `std::u8string` can be constructed from. This requires at least knowledge of the constructors, although an implementation could do heroics to depend pulling in the header.

Ignoring the dependencies, the proposed design has multiple other problems that could be fixed:

* The `name` member, if present, stores a copy of a user-provided string, which is then further copied into the compiler internal storage that represents a data member specification when calling `data_member_spec`.
* It is an error to specify both `alignment` and `width` yet the API does nothing to prevent that.
* Adding additional options is a potential ABI break if there are runtime functions that take or return a `data_member_options_t` (which is weird, but possible).

A design that instead provides multiple creation functions combined with setters has none of those problems.

## Proposed Design

```cpp
class data_member_spec_t {
public:
    consteval data_member_spec_t& name(/* depends on polls 4.1 and 1.3 */ name); // set the name
    consteval data_member_spec_t& no_unique_address(bool enable = true); // set no_unique_address

    consteval operator info() const; // build the data member specification
};

consteval data_member_spec_t data_member_spec(info type); // unnamed, unaligned member with no attributes
consteval data_member_spec_t data_member_spec_aligned(info type, int alignment; // unnamed, aligned member with no attributes
consteval data_member_spec_t data_member_spec_bitfield(info type, int width); // unnamed bitfield with no attributes
```

We provide named functions to create the three different cases of data members.
They return a builder object that can be further modified with setters and implicitly converted to an `info` object.

An implementation of `data_member_spec_t` that guards against ABI breaks can just store a single `info` object that represents the data already given,
and modifies the compiler internal representation when calling `.name()` and `.no_unique_address()`.

This requires also changes to `define_class` to accept a range of types convertible to `info`.

## User impact

The API changes dramatically, but it does not affect the convenience in any way.

::: cmptable

### Before

```cpp
define_class(^storage, {data_member_spec(^T, {.name = "foo", .no_unique_address = true})})
```

### After

```cpp
define_class(^storage, {data_member_spec(^T).name("foo").no_unique_address()})
```

:::

## Implementation impact

An implementation that does not care about minimizing dependencies can implement `data_member_spec_t` in terms of the current `data_member_options_t`.

## Wording

TBD

---
references:
  - id: P2996
    citation-label: P2996R7
    title: "Reflection for C++26"
    author:
      - family: Revzin
        given: Barry
      - family: Childers
        given: Wyatt
      - family: Dimov
        given: Peter
      - family: Sutton
        given: Andrew
      - family: Vali
        given: Faisal
      - family: Vandevoorde
        given: Daveed
      - family: Katz
        given: Dan
    URL: https://wg21.link/P2996R7
  - id: bloomberg-clang
    citation-label: bloomberg-clang
    title: "Bloomberg's clang implementation of P2996"
    URL: https://github.com/bloomberg/clang-p2996
  - id: prototype-impl
    citation-label: prototype-impl
    title: prototype implementation
    URL: https://gist.github.com/foonathan/457bd0073cfde568e446eb4d42ec87fe
  - id: daveed-keynote
    citation-label: daveed-keynote
    title: "Daveed Vandevoorde's closing CppCon2024 keynote"
    URL: http://vandevoorde.com/CppCon2024.pdf
  - id: daw_json_link
    citation-label: daw_json_link
    title: "daw_json_link"
    URL: https://github.com/beached/daw_json_link/blob/85f5f3f3d15a27fa000733f758b157d2267a74c8/include/daw/json/daw_json_reflection.h
  - id: lexy
    citation-label: lexy
    title: "lexy"
    URL: https://github.com/foonathan/lexy
---
