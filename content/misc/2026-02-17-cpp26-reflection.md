---
title: "C++26 Static Reflection: What P2996 Actually Gives Us"
date: 2026-02-17
draft: false
tags: [c++, reflection, metaprogramming, P2996]
---

> Slides: [Google Slides](https://bit.ly/cpp26reflection) ·  [PDF](../downloads/cpp26reflection.pdf)

"Can a string be a variable name?" That's the informal one-liner for reflection — the ability for a language to inspect and modify its own structure. C++26 is finally getting compile-time static reflection via P2996R13 and a constellation of companion proposals. This post walks through what the proposals actually provide, how they compare to pre-C++26 workarounds, and how other languages handle the same problem.

## What Static Reflection Means

Reflection, broadly, is a language's ability to introspect its own constructs: types, fields, methods, annotations, expressions. Dynamically typed or interpreted languages (Python, JavaScript) have had this trivially forever — at runtime, everything is inspectable. Java has had runtime reflection since JDK 1.1 (1997).

**Static** reflection is the restricted ability to do this at **compile time**, within a statically typed, ahead-of-time compiled language. The constraint is that all reflection must resolve during compilation — no runtime overhead, no type erasure, no dynamic dispatch. This is the version C++ is getting.

The immediate applications:

- **ORM** — automatically mapping struct fields to database columns without manual registration macros
- **Serialization/deserialization** — JSON, Protobuf, MessagePack marshalling derived from struct definitions
- **Struct of Arrays (SoA)** — transforming `AoS` layouts to `SoA` at compile time for cache-friendly access patterns
- **Code generation / transpilers** — programmatic code emission based on type structure

## Pre-C++26: What We Had

C++ has always had extremely limited runtime type information (RTTI):

- `typeid` gives you mangled type names
- `dynamic_cast` detects castability in inheritance hierarchies

That's it. You can't enumerate fields. You can't iterate over members. You can't reflect on function parameters. For anything beyond this, the community built elaborate workarounds:

- **Boost.PFR** — treats aggregates as tuples, allowing indexed access (`get<0>(person)` instead of `person.name`), but limited to ~200 fields and no access to field *names*
- **Boost.Hana** — introspection via macros (`BOOST_HANA_DEFINE_STRUCT`)
- **skypjack/meta** — manual type registration at runtime
- **Apache Thrift** — sidesteps the problem entirely with a custom IDL compiler

All of these are either limited, macro-heavy, or require a separate compilation step. P2996 makes them largely unnecessary.

## P2996R13: The Core Proposal

The central design decision: a **single opaque type** `std::meta::info` represents all reflected entities — types, variables, functions, namespaces, base classes, enumerators, templates. This is deliberate. A single type means reflected values compose naturally with standard algorithms and containers.

### The Two Operators: `^^` and `[: :]`

**Reflection operator `^^`** — lifts a compile-time entity into a `std::meta::info` value:

```cpp
constexpr std::meta::info r = ^^int;  // r represents the type 'int'
constexpr auto s = ^^std::string;     // s represents std::string
```

**Splice operator `[: :]`** — the inverse. Injects a `std::meta::info` back into code as the entity it represents:

```cpp
typename [:r:] x = 42;  // equivalent to: int x = 42;
```

The `typename` keyword is required when the compiler can't determine whether the spliced expression is a type.

### Key Introspection Functions

P2996 provides a rich set of `consteval` functions in `std::meta::`:

| Function | Purpose |
|---|---|
| `identifier_of(r)` | The name of the reflected entity as a string |
| `type_of(r)` | The type of a reflected variable/member |
| `parent_of(r)` | Enclosing scope/class |
| `members_of(r)` | All members of a class/namespace |
| `nonstatic_data_members_of(r)` | Non-static fields of a struct/class |
| `static_data_members_of(r)` | Static fields |
| `bases_of(r)` | Base classes |
| `enumerators_of(r)` | Enumerator values of an enum |
| `template_of(r)` / `template_arguments_of(r)` | Template introspection |
| `substitute(tmpl, args...)` | Construct a template specialization |
| `extract<T>(r)` | Extract the compile-time value from a `meta::info` |
| `define_aggregate(r, members)` | Programmatically define a struct |

### Access Control: `access_context`

Reflection respects C++ access control by default. `std::meta::access_context::current()` reflects only accessible members. To bypass access control (e.g., for serialization of private fields), use `std::meta::access_context::unchecked()`.

### Example: Struct to Struct-of-Arrays

One of the most compelling demonstrations — `define_aggregate` lets you programmatically define a new struct type at compile time:

```cpp
// Given:  struct Point { float x, y, z; };
// Produce: struct PointSoA { std::vector<float> x, y, z; };
```

This is done by iterating over `nonstatic_data_members_of(^^Point)`, wrapping each field's type in `std::vector<>` via `substitute`, and passing the result to `define_aggregate`. [Godbolt demo](https://godbolt.org/z/vqxzMoPcj).

## Companion Proposals

P2996 doesn't stand alone. Several companion papers fill in gaps:

**P3394R4 — Annotations for Reflection.** Attach arbitrary metadata to declarations that can be queried at compile time. Think Java's `@Annotations` or Rust's `#[derive()]` attributes, but for C++.

**P3293R3 — Splicing a Base Class Subobject.** Enables splicing base class references, needed for full reflection over inheritance hierarchies.

**P3491R3 — `define_static_{string,object,array}`.** A workaround for the lack of non-transient `constexpr` allocation. Lets you materialize compile-time computed strings and arrays as static storage duration objects.

**P1306R5 — Expansion Statements (`template for`).** Provides `template for` loops that expand at compile time over reflected members, eliminating the need for `std::apply` and fold expressions in many cases.

**P3096R12 — Function Parameter Reflection.** Extends reflection to function parameter names and types.

**P3560R2 — Error Handling in Reflection.** Structured error reporting when reflection operations fail.

## The Evolution of Tuple Printing: A Case Study

The progression from C++11 to C++26 for something as simple as printing a tuple illustrates what reflection buys you:

**C++11** — Recursive template specialization with SFINAE. Verbose, `O(N²)` compile time on old compilers, hard limit of ~1024 elements.

**C++14** — `std::integer_sequence` replaces recursion. Still needs helper functions and `std::initializer_list` tricks to emulate fold expressions.

**C++17** — `std::apply` + fold expressions. Cleaner, but still requires unpacking machinery.

**C++26** — `template for` over `std::tuple` elements. The entire implementation collapses to a direct loop:

```cpp
template <typename... Ts>
void print_tuple(const std::tuple<Ts...>& t) {
    template for (auto elem : t) {
        std::cout << elem << " ";
    }
}
```

No recursion, no integer sequences, no `std::apply`, no fold expressions.

## Reflection in Other Languages

### Rust

Rust does *not* have static reflection in the traditional sense. However, procedural macros (`proc_macro`) can read and manipulate the AST at compile time. This is **syntactic** reflection — you operate on token streams, not semantic types. The `#[derive(Serialize)]` pattern is the canonical example. It's powerful but fundamentally different from C++26's semantic reflection, where you introspect fully resolved types with their layouts, access specifiers, and template arguments.

### Java

Full runtime reflection since JDK 1.1 — `Class.getFields()`, `Method.invoke()`, etc. JEP 416 and Project Babylon are exploring code reflection (the ability to reflect on method *bodies*, not just signatures). Java's reflection is runtime and carries performance costs; C++26's is entirely compile-time.

### Python & JavaScript

Interpreted languages with pervasive runtime reflection. `getattr()`, `dir()`, `inspect` in Python; `Object.keys()`, `Reflect` API in JavaScript. Everything is inspectable at runtime because there's no ahead-of-time compilation to erase type information.

## Getting Started Today

The Bloomberg P2996 compiler fork is the reference implementation:

```bash
git clone https://github.com/bloomberg/clang-p2996
# Build the compiler, then build libc++ with it (needed for <meta>)
llvm/build/bin/clang++ -std=c++26 -freflection-latest \
    -Ilibcxx/include -nostdinc++ -isystem libcxx/include \
    -Llibcxx/build/lib -lc++ -lm -lc -pthread \
    your_program.cc -o your_program
```

Or use [Godbolt](https://godbolt.org/) with the P2996 compiler for quick experiments.

## References

- [P2996R13: Reflection for C++26](https://isocpp.org/files/papers/P2996R13.html)
- [P1306R5: Expansion Statements](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p1306r5.html)
- [P3394R4: Annotations for Reflection](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3394r4.html)
- [P3491R3: define_static_{string,object,array}](https://isocpp.org/files/papers/P3491R3.html)
- [Bloomberg clang-p2996](https://github.com/bloomberg/clang-p2996)
- [Effective Rust: Reflection](https://effective-rust.com/reflection.html)
- [Java Reflection Tutorial](https://docs.oracle.com/javase/tutorial/reflect/)
- [Project Babylon (Java code reflection)](https://openjdk.org/projects/babylon/)
