# Basic Language Components
This is just a random, basic and certainly not complete section about various basic C++ language features and constructs.

- [Basic Language Components](#basic-language-components)
    - [ADL (Argument Dependent Lookup)](#adl-argument-dependent-lookup)

## ADL (Argument Dependent Lookup)
- Also known as _Koenig-Lookup_
- Technique to allow namespaces and free (non-member) functions to interplay nicely with types to implement the basic _Interface Principle_ ([Sutter](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0934r0.pdf)):
  - For a class `Foo`, all functions (including free functions), that both
    1. _mention_ `Foo` and
    2. are _supplied_ with `Foo` (are in the same namespace)

    are _logically_ part of `Foo`, because they form part of the __Interface of__ `Foo`. 

- Key ideas: 
  1. Keep a type and its non-member function interface in the same namespace
  2. Keep types and fucntions in separate namespaces unless they're sepcifically intended to work together.
   
- __The potential _error_ in ADL__: It pulls in __all__ functions in the namespace of the type, rather than only the functions that are part of the interface of the type (that that _mention_ the type): This is often a problem if there are unconstrained templated functions declared that will be pulled in and are a better match than the interface funcions for a given cv-qualified name.
  
```cpp
// Example with unspecified behavior
#include <vector>

namespace foo {
    using Bar = std::vector<Foo*>;

    // do some special copy on Bar
    void copy(const Bar&, Bar&, bool param);

    void doStuff(const Bar& bar) {
        // if vector includes algorithm 
        // std::copy might be a better match
        copy(bar, bar, false);
    }
}
```

  
