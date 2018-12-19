# C++/CLI
This article is just a short overview of C++/CLI functionalities to create interfaces between .NET and C++.

If you came to this page by accident (e.g. Google search) please be aware of the following details:

>This is just a private article to remind myself of various aspects of a topic. This article might contain typos, ill-formed code, errors and is certainly not a useful reference for everybody. A lot of code examples are not even tested in the case i got the information from a blog or book! The only reason this article exists is to give me easy access to information i gathered when learned about a certain topic.

__Table of Contents__:
- [C++/CLI](#ccli)
    - [What is CLI](#what-is-cli)
    - [Differences between C++ and C++/CLI](#differences-between-c-and-ccli)
    - [Object Lifetime](#object-lifetime)
        - [Destruction and Finalization](#destruction-and-finalization)
        - [Automatic objects (stack allocated)](#automatic-objects-stack-allocated)
    - [Inheritance](#inheritance)
    - [Interfaces](#interfaces)
    - [Values and References](#values-and-references)
        - [Structs](#structs)
        - [Enumerations](#enumerations)
    - [Exception Handling](#exception-handling)
        - [Throwing Exceptions](#throwing-exceptions)
        - [Handling Exceptions](#handling-exceptions)
        - [Finally block](#finally-block)
    - [Safe Casting](#safe-casting)
    - [How to use a C++/CLI dll in VS with another language?](#how-to-use-a-ccli-dll-in-vs-with-another-language)
    - [Arrays and Collections](#arrays-and-collections)
    - [Properties](#properties)
        - [Scalar Properties](#scalar-properties)
    - [Delegates and Events](#delegates-and-events)
        - [Delegates](#delegates)
        - [Events](#events)
    - [Managed vs. Unmanaged Code](#managed-vs-unmanaged-code)
        - [Mixed classes](#mixed-classes)
        - [Pinning and Boxing](#pinning-and-boxing)
        - [PInvoke to call functions in the Win32 API](#pinvoke-to-call-functions-in-the-win32-api)

## What is CLI

C++/CLI is a special version (extension) of C++ designed to run on the .NET Framework.

- There are certain limitations to pure C++ that you cannot do in C++/CLI
- There are certain extensions to work with the .NET runtime
- __CLR__: Common language runtime

## Differences between C++ and C++/CLI

- The sizes of basic types are always fixed size.
- There are not pointers since the CLR has garbage collection
  - The runtime will provides __handles__ (_tracking handles_) that are like garbage collected pointers: `Persion^ p;`
  - To create a garbage collected handle: `Person^ p = gcnew Persion("Foo")`
  - If the all handles to an object go out of scope the garbage collector will delete the object
  - Handles can be `nullptr`
- There is no implict copy constructor in C++/CLI. You have to provide one yourself.
- References: _Tracking references__ (since GC can relocate) using
    ```cpp
    MyFoo(const MyFoo% other); // in C++: MyFoo(const MyFoo& other);
    ```
- No default arguments on managed types
- No converting constructors
- The `#using <library.dll>` preprocessor directive imports the Microsoft Intermediate Lange (MSIL) file `library.dll`. After import it is possbile to use __managed__ data and constructs defined in this library file (e.g. from `mscorelib.dll`).

## Object Lifetime

### Destruction and Finalization

- __Finalization__: What happens when the GC cleans an object up
- Destructor:
    ```cpp
    Person^ p = gcnew Person("foo");
    ...
    delete p; // will call the destructor of p
    ```
- Finalizers:
  - Will be called if the GC is cleaning up a object
  - You need a finalizer if you handle unmanaged code in your object
  - Example:
    ```cpp
    ref class Foo {
        public:
            ~Foo(); // destructor
            !Foo(); // finalizer
    };
    ```
  - Only declare a finalizer if it is needed (GC has to run it even if it is empty)
  - There is no guarantee in which order finalizers run
  - Finalizers are __not called__ during application termination for objects that are still alive!
  - If the object was created and is still alive and the application returns from e.g. main __only the finalizer of the object is called__.
- Destructors are only called on explicit `delete` - __no finalizer will run then__! If you need a finalizer you __should call it from the destructor__.

### Automatic objects (stack allocated)

- Automatic objects (stack allocated):
  - Under the hood there is no stack and objects have still _handles_. If an automatic object goes out of scope the destructor is called!
- Copy constructor:
    ```cpp
    ...
    MyFoo(const MyFoo% other);
    ...
    MyFoo^ foo = gcnew MyFoo();
    MyFoo% ref_foo = *foo; // Tracking reference to foo
    MyFoo cpy_foo = *foo;  // Calls the copy constructor and cpy_foo lives on the stack
    ```

## Inheritance

- Only __single and public__ inheritance allowed
- Syntax: `ref class Foo : Base {...};`
- Every object implicitly inherits from `System::Object` in .NET
- Abstract classes work by using the `abstract` keyword:
    ```cpp
    ref class AbstractBase abstract {...};
    ```
- You can _seal_ a class (like final in C++) so that it is not allowed as a base class:
    ```cpp
    ref class NoBase sealed {...};
    ```

## Interfaces

- Like a base class with only pure virtual functions
    ```cpp
    interface class Inter {
        void foo();
    };

    ref class Bar : Inter
    {
        public:
            virtual void foo(); // must be virtual to implement the interface
    }
    ```
- An interface cannot contain data members.
- All members of an interface are public and abstract by default.
- Interface names should start with an capital `I` by convention
- A class can only inherit from one base but can implement as many interfaces as desired

## Values and References

- C++/CLI types are just aliases to the __boxed .NET__ types:
    ```cpp
    int n   = 0;    // managed C++ type
    Int32 x = 0;    // use .NET native type -> means the same thing as n
    ```
- Properties:
  - All value types inherit from `System::ValueType`
  - Are stored on the stack and not on the runtime heap
  - Are not garbage collected
  - Always direct access and not via references/pointers
  - Copying is real copying
  - Cannot be used as base classes
  - Difference to build in types: Can be treated as an object
  - You can add your own value types : `structs` (managed structs)

### Structs

```cpp
value struct S {    // value is important to distinguish 
                    // between c++ structs and .NET structs
    int x;
    int y;
};

S s;            // uses the stack, no need for gcnew
s.x =10;
```

- Differnces to classes:
  - You cannot override the default constructor
  - Cannot have destructors or finalizers (are not GCed)
  - Can implement interfaces
  - Cannot use inheritance
  - Can have constructors

### Enumerations

- Named integer constants
- Are value types
- Synatax:

```cpp
    // like class enums in C++
    public enum class Foo : int {
    // public/private is needed to distinguish to C++11 class enums
    // the underlying type is optional
        Bar = 1, // starts with 1 and not 0 (default)
        Bazz,
        Boo
    };
```

- Can be casted to an `int`

## Exception Handling

- In .NET exceptions can be used accross languages (e.g. throw in C++/CLI and catch in VB .NET)
- Three different types of exceptions:
  - __Traditional C++ exceptions__
  - __C++/CLI exceptions__
  - __Windows Structured Exception Handling__ (SEH)
- You can mix C++ exceptions and managed exceptions inc C++/CLI
- `Finally` block
- If an exception not derived from `System::Exception` escapes from C++/CLI to non-C++ code (language) the exception is wrapped into a `RuntimeWrappedException`.

### Throwing Exceptions

- In C++: Typically throw and catch by reference
- In .NET: Typically throw objects derived from `System::Exception`
- Example:

```cpp
throw gcnew System::ArgumentException("Heeelp");
```

### Handling Exceptions

- Use a `try catch` block like in C++:

```cpp
try {...} catch(MyException^ e) { ... }
```

### Finally block

- Cleanup code after an exception is thrown

```cpp
try {...}
catch(MyException^ e) { ... }
finally{...} // executed after the catch block
```

## Safe Casting

- New keyword `safe_cast<T^>(handle)` as a safe casting alternative.
- Throws `InvalidCastException ^` at runtime for a bad cast.
- Basically a `dynamic_cast` replacement that throws.

## How to use a C++/CLI dll in VS with another language?

- Build dll with `/clr` option
- Create e.g. a VB.NET app and add a reference to the dll.

## Arrays and Collections

- Managed arrays: Equivalent of C++ arrays but lives on the managed heap
- Examples:

```cpp
array<int>^ a1;     // 1D array of integers
array<int, 2>^ a2;  // 2D array of integers
array<Foo^> a3;     // 1D array of Foo handles

a1 = gcnew array<int>(10);          // 10 element array
a1 = gcnew array<int>(3) {1,2,3};   // 3 element array initialized
array<int>^ a_new = {1,2,3};        // short form
```

- `for each` loop: Works with every Collection that implements `IEnumerable`.

```cpp
for each (int a in a1)
{
    ...
}
```

- Enumerators are the .NET implementation of Iterators. Enumerators are immutable (you cannot write through an Enumerator)!

- Other Collection classes:
  - `Dictionary<K,V>`: Key/value hashtable
  - `HashSet<T>`: A set (a unique collection)
  - `LikedList<T>`: AQ doubly-linked list
  - `List<T>`: Non-fixed size array (expandaple)
  - `Queue<T>`: A queue
  - `SortedList<K,V>`: Get elements by index of by key
  - `Stack<T>`: Push/pop stack
  
- The STL/CLR library:
  - CLR version of the C++ STL and live in `<cliext/...>`
  - They can hold managed types
  - Totally separate from C++ STL
  - Just for legacy support (DO NOT USE IT)!

## Properties

- Supported by the MSIL (Microsoft Intermediate Language) -> supported in .NET languages
- Properties are like virtual data members with syntactic suggar:
- Can be (pure) `virtual` even with separation on get and set

```cpp
Foo^ foo = gcnew Foo();
foo->Name = "foo";      // calls the setter
name = foo->Name;       // calls the getter
```

### Scalar Properties

- Access to a single value

```cpp
ref class Foo {
    String^ name;
    public:
        ... // constructor and stuff
        property String^ Name       // Capitalize Name by convention
        {
            String^ get() { return name; }      // if ommited -> write only
            void set(String^ n) { name = n; }   // if ommited -> read only
        }
};
```

- Indexed: Let properties get accessed like arrays

```cpp
ref class Foo {
    array<int>^ a;
    public:
        ... // constructor and stuff
        property int A[long]       
        {
            int get(long idx) { return a[idx]; }     
            void set(long idx, int value) { a[idx] = value; }  
        }
};
```

## Delegates and Events

Delegates are the .NET equivalent of function pointers and form the basis of the .NET event handling mechanism.

### Delegates

A delegate is a class whose purpose it is to invoke one or more methods that have a particular signature (indirect call to execute a function).

- It is possible to attach more than one method to a delegate.
- All the functions are called in order when the depegate is invoked.
- Base classes for delegates in .NET: `System::Delegate` (for single methods) and `System::MulticastDelegate` for calling more than one method. In C++/CLI __all delegates are multicast delegates__.

```C++
// define a delegate type "NumericOperation" that implicitly inherits 
// from System::MulticastDelegate
// Can be bound to any function with the signature double(*)(double)
delegate double NumericOperation(double);

ref class Foo
{
public:
    static double SquareMe(double d) {
        return d*d;
    }

    double CubeMe(double v) {
        return v*v*v;
    }
};
...
// e.g. code in main
// ctor takes just the address of the function to call
NumericOperation^ operation = gcnew NumericOperation(&Foo::Square);

double sq_fp   = operation->Invoke(5.0); // should be 25.0
double sq_fp_2 = operation(5.0); // implicit call to  Invoke

Foo^ foo = gcnew Foo();
// bind to non-static memeber functions
NumericOperation^ op = gcnew NumericOperation(foo, &Foo::CubeMe);

double cuby = op(3.0); // like calling foo->CubeMe(3.0);
```

- You cannot use a delegate to call unmanaged code or code that is defined in the global namespace.

### Events

- In .NET events are implemented as a publish-and-subscribe mechanism.
- Events are based on delegates: An event source declares delegate for each event that it wants to generate and event receivers define suitable methods that are passed to the source.
- When an event fires it invokes all the subscribed functions.
- Event properties:
  - Can only be fired by the type that declares it
  - Clients can only add and remove event handler functions using `+=` and `-=` to prevent resetting the invocation list.
- The `event` keyword:
  - Compiler will generate code to implement the underlying delegate meachanism.
  - For event `AEvent` the following public/protected methods will be generated:
    - `add_OnAEvent` that calls `Delegate::Combine` to add a receiver to the event's invocation list. Shortcut: `+=` on the event object itself.
    - `remove_OnAEvent` that calls `Delegate::Remove` to remove a reveiver from the event's invocation list. Shortcut: `-=` on the event object itself.
    - `raise_OnAEvent` that  is a protected method that calls `Delegate::Invoke` to call all methods on this event's invocation list.

```cpp
// Event source delegates
// This is the signature event receivers must implement
delegate void EventHandlerA(String^ str);
delegate void EventHandlerB(String^ str);

// Event Source class
ref class Source {
public:
    // the events (one declaration for each event you want to raise)
    event EventHandlerA^ OnAEvent;
    event EventHandlerB^ OnBEvent;

    // event raising functions
    void RaiseA(String^ msg) {
        OnAEvent(msg);
    }

     void RaiseB(String^ msg) {
        OnBEvent(msg);
    }
};

// Event receiver that listens for events
ref class Listener {
    Source^ source;

public:
    Listener(Source^ src) {
            source = src;
            // add our handlers
            source->OnAEvent += gcnew EventHandlerA(this, &Listener::EventA);
            source->OnBEvent += gcnew EventHandlerA(this, &Listener::EventB);
    }

    // handler methods
    void EventA(String ^msg) {
        Console::WriteLine("Event A, message is {0}", msg);
    }

    void EventB(String ^msg) {
        Console::WriteLine("Event B, message is {0}", msg);
    }

    void RemoveHandler() {
        // unsub only for EventA
        source->OnAEvent -= gcnew EventHandlerA(this, &Listener::EventA);
    }
};

// setup source and listener
Source ^src   = gcnew Source();
Listener ^rcv = gcnew Listener(src);

// Fire events
Console::WriteLine("Fire both events:");
src->RaiseA("Foo");
src->RaiseB("Bar");

```  

- Standard events and System::EventHandler
  - All standard .NET handlers have the following signature:

```cpp
  // src:  Reference Object that raised the event
  // args: Reference to object of base type EventArgs (extra infos about the event)
  void MyPersonalHandler(Object^ src, EventArgs^ args); 
```

- The programmer could use the `System::EventHandler` delegate instead of defining new delegates for event handling.

```cpp
ref class Foo {
    int counter;
    int limit;

public:
    event EventHandler^ LimitReached; // use of System::EventHandler

    Counter(int lim) : 
        limit(lim), 
        counter(0) {
    }

    void inc() {
        ++count;
        if (count == limit) {
            LimitReached(this, gcnew EventArgs()); // raise event
        }
    }
};

ref class Bar {
public:
    static void CallMe(Object^ src, EventArgs^ args) {
        Console::WriteLine("Limit reached");
    }
};

...
Counter count(3);
count.LimitReached += gcnew EventHandler(&Bar::CallMe);

```

## Managed vs. Unmanaged Code

### Mixed classes

- It is possible to mix managed and unmanaged types as members
- Must use pointers to unmanaged objects
- Handles do not work
- `GCHandle`:
  - Handle from unmanaged code to managed
  - Can be passed to unmanaged code
  - Get handle by using `Alloc` and release handle by calling `Free`
    - Easier with helper template:
    ```cpp
    #include <gcroot.h>
    using namespace System::Runtime::InteropServices;
    class CppClass {
        gcroot<ManagedClass^> handle; // uses RAII to release the handle
        public:
            CPPClass(gcroot<ManagedClass^> m) : handle(m) {};
    };
    ...
    ManagedClass^ m = gcnew ManagedClass();
    CppClass cpp_class(m);      // implicit conversion
    ```
    - Use `auto_gcroot` to let the RAII call the destructor

- Remember the difference between destruction and a finalizer!
  - Dispose pattern:
  - In the Destructor: Clean up managed and unmanaged objects
  - In the Finalizer: Clean up __only__ unmanaged objects
  - If delete is explicitly called: Call the destructor and supress the call of the Finalizer
  - If not -> Finalizer will run at GC time (call the finalizer in the destructor)
  - Important:
    - Calling the Finalizer is permitted -> Possibility that the Finalizer runs multiple times
    - Use a check for `nullptr`!
    - Invoking the Destructor is optional
    - A Finalizer will not call the Finalizer of a base class by default!
      - Call the Finalizer from the Destructor
      - Clean up unmanaged code in the Finalizer but check for double deletion
      - Make Finalizer `protected`

### Pinning and Boxing

- __Interior pointers__: A pointer that address changes if a managed object is moved in memory by the GC. You cannot point to a whole managed object, __only to one field__.

- __Pinning pointer__: Pointer to a managed object but with a fixed value (GC cannot move object) around. Pinning a member of an object will pin the whole object.

```cpp
void cpp_fun(int* p); // c++ function taking a pointer

array<int>^ arr = gcnew array<int>(5);
pin_ptr<int> pin = &arr[2]; // pin the array

cpp_fun(pin); // implicit conversion
pin = nullptr; // zero out to unpin
```

- __Boxing and unboxing__:
  - Makes it possible for value types to be treated as objects.
  - __Boxing__ wraps a value type in an object _box_ so it can be used where an object reference is needed. In C++/CLI boxing is done automatically. What happens when an object is boxed?
        - A managed object is created on the CLR heap
        - The value of the type is copied, bit by bit, into the managed object
        - The address of the managed object is returned
  - Be aware that the managed object contains a __copy_ of the original object and changes will not be written back to the original value.
  - __Unboxing__ is done with a `safe_cast<T>`:
    ```cpp
    int i = 12;         // value type
    Object^ obj = i;    // automatically boxed (copied on the heap)
    int j = safe_cast<int>(obj); // unboxed (copied from the heap)
    ```

### PInvoke to call functions in the Win32 API

- Can be used to call stuff from a _naked_ dll
- Could work with code without .NET support
- Call Win API functions with no .NET implementation
- Method to do all that: `P/Invoke` (Platform Invoke)
  - Let you call functions in dlls
  - Underlying marshalling principle in C++/CLI (e.g. from `std::string` to `String^`)

```cpp
// call a MessageBox function from the WIN API
// Function is found in User32.dll

using namespace System::Runtime::InteropServices;
// P/Invoke prototype for the MessageBox function
// CharSet sets the type of strings to use
[DllImport("User32.dll", CharSet=CharSet::Auto)]
int MessageBox(IntPtr hwnd, String^ text, String^ caption, unsigned int type);

String^ text = "Some text";
String^ cap = "Caption";
MessageBox(IntPtr::Zero, text, cap, 0);

// User another name for your internal prototype
[DllImport("User32.dll", EntryPoint="MessageBox", CharSet=CharSet::Auto)]
int WindowsMessageBox(IntPtr hwnd, String ^text, String ^caption, unsigned int type);
```