# C++ 17 Features

This article is just a short overview of C++ 17 features.

- [C++ 17 Features](#c-17-features)
  - [Basic Language Features](#basic-language-features)
    - [Structured Bindings](#structured-bindings)
      - [Qualifiers](#qualifiers)
      - [Move Semantics](#move-semantics)


## Basic Language Features

### Structured Bindings

- Allows to initialize __multiple entities__ by the elements or members of an object.

```cpp
struct Foo {
    int a;
    std::string s;
};

Foo foo;
auto [u, v] = foo; // structured binding to bind members to new names

// return and bind
Foo getFoo() {
    return Foo{60, "foo"s};
}

auto [n, m] = getFoo();
```

- Best use to indicate semantic meaning on unnamed structures:

```cpp
for (const auto& [key, value] : mymap) {
    std::cout << key << " : " << value << "\n";
}
```

- As for a lot of new features in C++ (e.g. lambdas) there is some magic going on onder the hood and the compiler constructs some stuff for us:

```cpp
auto [u,v] = foo;

// -> this is what the compiler actually does
// in this case -> copy foo
auto compiler_gen_name = foo;
<ALIAS> u = compiler_gen_name.a;
<ALIAS> v = compiler_gen_name.s;
```

- The new names are aliases to members of a compiler generated variable. They are __not references__.
- We cannot access this new compiler-generated variable.
- The compiler generated variable will live as long a structured binding exists.

#### Qualifiers
- What happens when we use __qualifiers__? They (again) apply to the _magic_ anonymous entity the compiler created.
  
```cpp
// u and v reference to foo.i and foo.s
const auto& [u,v] = foo;
// compiler might construct something like this
const auto& compiler_gen_name = foo; // reference to foo
<ALIAS> u = compiler_gen_name.a;
<ALIAS> v = compiler_gen_name.s;

// lifetime extension rules for temporaries apply here as well
const auto& [x, y] =  getFoo(); 
```

- Important: Qualifiers apply to the _magic_ entity. They do not necessary apply to the aliased names. In the example from above, `decltype(u)` is `int` and not `const int&`.
- For the same reason, structured bindings __do not decay__ (type conversion when arguments are passed by value: arrays -> pointers, top level qualifiers are ignored).

```cpp
struct Foo {
    const int a[5];
    const int b[2];
};

Foo foo{};
auto [a,b] = foo;   // and b are both of array type
auto d      = a;    // d is of decayed type (int*)
```

[Wandbox](https://wandbox.org/permlink/fLvBKz2cKY2BYyjc)

#### Move Semantics

```cpp
struct Foo {
    int a;
    std::string s;
};

Foo foo;
// just a innocent rvalue reference (nothing moved yet)
auto&& [u,v] = std::move(foo);

Foo bar {10, "Hallo"s};
auto [u, v] = std::move(bar); // now we really moved to the magic entity
```

[comment]: # (Page 8 1.2)