# Effective C++

Class: C++

Created: Jun 4, 2020 6:28 AM

Finished: Jun 30, 2020

Type: Book

# Introduction

C++ offers too much. To write effective C++, you need to

- make design decisions, from big (class hierarchy) to small (pass-by-reference or pass-by-pointer)
- make those decisions happen. It's not easy.

# Accustoming to C++

## 1 C++ is a federation of languages

At least 4 sub-languages:

- **C.** Classic. No templates, exceptions, overloading, etc.
- **Object-Oriented C++.** C with Classes. Encapsulation, inheritance, polymorphism, virtual functions, dynamic binding. Apply traditional object-oriented design.
- **Template C++**. Generic programming. A different set of rules. Fret not.
- **The STL.** Has particular ways of doing things. Follow conventions.

Examples:

- Pass-by-value is efficient in C. Not for OOC++ and Template C++. But for STL with iterators, the rules apply again.

## 2 `const`, `enum`, `inline`> `#define`

Or, prefer compiler to preprocessor.

- Note: now to replace macro constants use `constexpr` instead of just `const`.
- *The enum hack*: uses a enum in a class to simulate `#define`. It's behaviour is more close to `#define` so it's widely used.
- For macro functions, use templates, `constexpr` and `inline`.

## 3 Use `const` whenever possible

### ROT

- declaring `const` helps compilers to find usage errors
- `const` can be applied to objects at any scope, function parameters, return types, and member functions as a whole
- for member functions, compilers check bitwise but you should use logical
- for duplication in versions of member functions, let the non-`const` call the `const`

### raw pointers:

- `const` on the left applies to the target type; `const` on the right applies to the pointer itself.
- `const Widget *pw` and `Widget const *pw` are equivalent, as the `const` appears on the left of the star.

### iterators

- 2 places to apply `const`
- `const T<S>::iterator` declares the iterator itself constant (very naturally! It's no different to `const int`).
- `T<S>::const_iterator` disables modifying the target element.

### functions

- 3 places to apply `const` (return type, params, whole function if member function)
- Sometimes it's necessary to return `const` types to disable assigning something to the returned value
- `const` member functions
    - For a const object, only `const` member functions can be invoked
    - You can overload functions whose signatures only differ in `const`ness
    - Two philosophies for deciding whether a member function should be `const`
        - Bitwise: member funcitons should be `const` iff they don't modify data members. This is how the compiler thinks by default.
        - **Logical**: member functions should be `const` iff it doesn't modify any member the client can see.
            - e.g. it's OK to update internal cache, even when the object is `const`.
            - To make compilers happy, use `mutable` when declaring fields than can be modified by `const` member functions.
    - To reduce boilerplate code between `const` and non-`const` version of the exact same function, use casts so non-`const`calls `const`.

        Note that don't do it the other way around: it might break the promise that `const` member functions must not modify the object.

        ```cpp
        const T& f() const {
            return something_complicated();
        }
        T& f() {
            return const_cast<T&>(std::as_const(*this).f());
        }
        ```

## 4 Ensure init before use

### ROT

- reading uninitialized values is UB. Thus, always initialize objects before use.
- manually initialize built-in type objects
- in ctor, prefer member init'n list to assignment inside body.
- list data members in the same order they're declared in class
- replace non-local static objects with local static objects

### init'n for built-in and user-defined types

- For non-member objects of built-in types (like a global `int` or `double`), it must be done manually.
- For almost everything else, the responsibility for init falls on constructors. Must ensure ctors you write initialize all members.
    - *Don't **assign*** to members in ctor body (which calls the default ctor and immediately copy-assigns a new value); use the **member** **initialization list** for efficiency, since only one copy construction happens.
        - For data members that are `const`, you can't even assign them.
    - Even for default ctors that should default initialize all members, explicitly provide a member init'n list, which should call default ctors for all user-defined type members and provide values for built-in type members.
    - If listing out all members in member init'n list is too repetitive, then consider using a pseudo-init'n function that is called by all ctors. Note that this still ensures you don't accidentally leave out any member.
- Initialization is not necessarily done by using `=` (copy or move construction). Things like `std::cin >>` also work.

### order of init'n

- Members of an object are init'ed in the same order as they are declared.
    - Different orders in the member init'n list doesn't affect this rule. But to avoid confusion you should ensure that they are the same.
- **Terms:**
    - A *static object* exists from the time it's constructed until the end of the program.
        - No stack or heap objects
        - global / namespace scope / `static` inside class / `static` inside functions / `static` at file scope variables are *static objects*.
        - `static` variable in functions are *local static objects*. Others are *non-local static objects*.
        - When the program exits (i.e. `main` returns), dtor static objects
    - A *translation unit* is the code that generates a single object file, usually a single source file plus all its headers.

Say we have two separately-compiled source files. Each one has some non-local static objects. Then if they reference each other **across translation units**, they might be using uninitialized data. Because...

The relative order of init'n of non-local static objects defined in different translation units is undefined.

To solve this, put the static object into a function.

```cpp
// before
extern FileSystem tfs;

// after
FileSystem& tfs() {
  static FileSystem fs;
  return fs;
}
```

- Note that if a variable in a function is `static`, you can return its reference. This can be confusing as they are "local" static objects.
- It's perfect for inlining.
- However, it's bad for multithreading, just like other non-`const` static objects. Try to move the static object into the master thread.

# Ctors, Dtors and `operator=`

## 5 Know default behaviour

If nothing's present, compiler will declare a copy ctor, a copy assgm oprt, and a dtor.

- Only generated when there are needed (used in the rest of the program)
- All are `public` and `inline`.

If you write ctors that **have parameters**, the compiler won't create a default ctor for you. However, the compiler still generates copy ctor / copy assgm oprt if missing and needed.

If you want to support copy assgm in a class containing a **reference / `const` member**, you must define the copy assgm oprt yourself.

## 6 Disallow generated functions you don't want

Use `= delete` on copy constructors to disable copying.

## 7 Make dtors virtual in polymorphic base classes

### reason

UB when a derived class object is deleted through a pointer to a base class with a non-`virtual` dtor. Reason: the base class non-`virtual` dtor will not call the proper dtor in the derived class and will likely only delete a part of the object. [See Random Bits](https://www.notion.so/C-Random-bits-e4aed87017ab4640b37863403d30df52#daac8dbfcfd744b0b3c99e52a2a8b61e).

Don't inherit from a class that doesn't have virtual dtors, like STL containers and `string`. Check before inherit. Compiler doesn't enforce.

However, you shouldn't declare every dtor virtual because

- It greatly increases the size of the object
- It makes the object incompatible with C

Only polymorphic base classes need pure virtual ctors. If a class shouldn't be a base class (STL containers etc.) or shouldn't be used polymorphically (an interface/protocol), it shouldn't have virtual destructors.

### pure virtual dtors

If you want a class to be an abstract class but you don't have any pure virtual functions (i.e. you have default implementation for all virtual functions), simply **define** a pure virtual dtor.

You must supply a definition for the dtor, possibly with `T::~T() = default`.

## 8 Prevent exceptions from leaving dtors

### ROT

- dtors must not emit exceptions
- if func called in dtor might throw, dtor should catch and terminate/swallow
- empower the client to react to the exceptions

### reason

You may have two exceptions active. UB.

### problem & solutions

It's easy not to throw exceptions that aren't catched, but when the dtor needs to release other resources by calling other functions, it may not be so obvious. Two ways to dogde:

- Terminate the program, typically by calling `std::abort`. It stops the current dtor from propagating the exception.
- Swallow the exception. Generally not a good idea, but if aborting leads to premature termination you need to swallow. Do recover.

A better strategy that moves exception handling code out of dtors and wraps those treatments:

- Lift the exception-throwing logic to another member function, exposing it to the client.
- Keep track of whether the client has called that function. If not, call it in dtor.

This enables the client to handle the exception by themselves; if they choose not to, the default behaviour will also be safe.

## 9 Never call virtual funcs during constr'n or destr'n.

### ROT

never call virtual funcs in ctor and dtor; they never go to a more derived class.

### reason

Say we have a base class (which has a virtual member function and whose ctor calls that function) and a derived class.

We declare an object of the derived class. It will call the ctor of the base class, which calls that virtual func. But since it's inside the base class's ctor, the base class version of the virtual func is called, not the derived version. So, during base class constr'n, virtual funcs never go down to the derived classes.

### object type treatment model

When constructing a derived obj, the base ctor will be called first. During that, the obj is deemed a base object by runtime type info utilities.

When destructing a derived obj, the base dtor will finally be called. The extra members from the derived class should be handled by the derived dtors already at that point, so during dtor of base class, the obj is deemed a base obj.

### problem & solutions

Like [Item 8](https://www.notion.so/Effective-C-73e0037beb024637b5e5e08848f8d4fd#b3ea10ba0ff2483db84d9c84ac960040), it's easy to spot direct violation but not virtual func calls in nested funcs called by c/dtors.

Make the virtual func non-virtual; pass polymorphic info needed by derived classes to the ctors of the base class, to treat subclasses differently.

## 10 Have assignment operators return a reference to `*this`

### ROT

This is for supporting chaining `=` operations.

```cpp
Widget& operator=(const Widget& rhs) {
  // ...
  return *this;
}
```

Also include different signatures and other assignment operators.

## 11 Handle assignment to self in `operator=`

### ROT

- make sure `operator=` is well-behaved even when rhs is same obj
- techniques: compare identity, reorder, copy-and-swap
- make sure any function operating on two objs or more work well even when some of the parameters are the same obj

### problem

Sometimes client will assign an object to itself, explicitly or implicitly, i.e. `obj = obj`. When that happens, the `operator=`'s rhs is the same object as `*this`, so normal logic that assumes lhs ≠ rhs can be disaster.

### solutions

- Add a guard: `if (this == &rhs) return *this;`
    - This is straightforward but it adds a test to every assignment
- Aim to make the `operator=` exception-safe.
    - e.g. don't `delete` the original member before you're sure the `new`ed member works
    - imagine if rhs is the obj itself and see what could go wrong
- Use the "copy and swap" technique
    - define a `swap` function for the class
    - in `operator=` implementation, copy-construct a new object, swap, and `return *this;`
        - alternatively, use the naked type as the type of rhs (which triggers copy constr'n) and just swap. Might be better for optimization

## 12 Copy all parts of an object

### ROT

- copy functions must copy all of an obj's data members and all of its base class parts
- don't implement one of the copying functions in terms the other
    - use 3rd func that both call

### potential partial init'n

The compiler-generated default copy ctors and copy assignment operators will copy all members; but if you choose to write copying functions by hand, you need to make sure every member is in the member init'n list.

When adding a new member to a class, also update all copying functions.

### init'n of subclasses

Invoke base class's copying functions in subclass's copying functions

- For copy ctor, invoke `BaseClass(rhs)` in member init'n list
- For copy assignment operator, call `BaseClass::operator=(rhs)` in `operator=` implementation

### boilerplate

Extract a common func that both copying funcs call. Usually `private`, named `init`.

# Resource Management

Using the ad hoc way to deal with resource management is error-prone and weak to change. We need better methods.

## 13 Use objects to manage resources

We might have an obj that must release resources by `delete` or other functions when it's done being used. Ensuring that by hand is impossible as code has branches and changes over time. Thus we wrap that obj in another obj...

- whose ctor aquires all resources as part of init'n (RAII)
- whose dtor will do all the releasing

You don't have to write all resource-managing objs by yourself. If all resources you need to manage is dynamically allocated memory, use `[unique_ptr](https://www.notion.so/C-Random-bits-e4aed87017ab4640b37863403d30df52#2c2deb8e4169407dad2f8d888b7fba89)` and `[shared_ptr](https://www.notion.so/C-Random-bits-e4aed87017ab4640b37863403d30df52#83edf3f44b824f728a326ab1460f7655)`. (both them work for C-arrays)

## 14 Think copying behavior in resource-managing classes

You must think about when writing an RAII class: what should happen when an RAII object is copied?

### prohibit copying

**Scenario**: it makes no sense to allow copying

Just prohibit it by setting the copying functions `= delete;`

### reference-count the underlying resource

**Scenario**: it's OK for multiple objects to use one resource, and you want resources to be released when the last of such objects is destroyed.

Then you must add a counter in the RAII class. Of course, when copying an RAII object, increase the counter by 1. Check for the counter when destroying.

If you don't want to manually write a class for the reference-counting behavior, `shared_ptr` allows custom deleter, if the resource is not just memory and can't be simply `delete`d. Just pass the function to be called as the second param.

### copy the underlying resource

**Scenario**: you can have as many copies of a resource as you like, as long as each copy will release the resource properly.

Then you perform a "deep copy," i.e. create a new copy with everything the same. This is very common for classes that represent a slice of data, like a `string`.

### transfer ownership of the underlying resource

This is rare. You invalidate the previous obj and make the new obj usable, like `auto_ptr`.

## 15 Provide access to raw resources in res-mng class

### ROT

- define funcs to get internals
- think about whether to allow implicit conversion

### reason

There might be cases (interaction with other APIs) where the user needs to use the wrapped resource (usually the pointer) inside an RAII class.

### in practice

- implement a `.get()` method that returns the internal resource
- ☢️DANGER: define implicit conversion operator if it's so common
- for pointers, overload `->` and `*`

## 16 Use matched `[]` in `new` and `delete`

Wrong use leads to UB.

Sometimes it's hard to tell if the thing to be `delete`d is a C-array, like when the array is declared in a class or you have `typedef`ed an array into something.

## 17 Store `new`ed objs in smart ptrs in standalone statements

### ROT

When creating a smart pointer, don't put them in func calls.

### scenario

Say we have a function that takes a `shared_ptr` and an `int`. We might want to call it like this, in one line, gracefully:

```cpp
void f(shared_ptr<T> spt, int i);
f(shared_ptr<T>(new T), g());
```

Surprisingly, this can leak. For `f` to be called, the two arguments must be computed:

- Call the ctor `shared_ptr` with the `new`ed `T`.
- Call `g`.

The first task can be break down into two sub-tasks, and there are three things to do. Note that the compiler can do the three tasks in random order. Suppose the compiler decides this order:

- Execute `new T`.
- Call `g`.
- Call `shared_ptr`.

Disaster will happen if the call of `g` goes wrong and throws exception. The pointer from `new T` will be lost and cause leak.

### problem

If an exception happens between "the time after an object is `new`ed" and "it's wrapped in a smart pointer", it leaks.

### solution

Extract out creation of smart pointers.

```cpp
shared_ptr<T> spt(new T);
f(spt, g());
```

The `make_` funcs in later C++ might solve this problem because it won't use `new`.

# Designs and Declarations

Designing software is about designing interfaces. Function, class, template interfaces.

## 18 Make interfaces easy to use right and hard to use wrong.

Ideally, "it does what client expects" and "it compiles" should be equivalent.

### ROT

To facilitate correct use

- consistency in interfaces
- behavioral compatibility with built-in types

To reduce errors

- create new types
- restrict operations on types
- constrain obj values
- eliminate client-side res-mng responsibilities

### anticipate mistakes

This requires you to think about what mistakes the client can make. If you want to stop the mistakes at compile time, you may turn to the type system.

Like for preventing inputting invalid dates represented by `int`s, make separate classes for Day, Month and Time, and enforce checking.

### establish consistency among built-in types and other similar types

...because clients know how built-in types behave, and if they learned one thing, there's less friction to use the other thing correctly.

### reduce things that the client needs to remember

Especially returning raw pointers. Use smart pointers with a custom deleter [(#14](https://www.notion.so/nerrons/Effective-C-73e0037beb024637b5e5e08848f8d4fd#86d3a6139a3249eba4efd0e7d50c427e)).

## 19 Treat class design as type design

You need to understand the issues you face that led you to thinking about building a new class.

- How to create and destroy?
    - Affects ctor, dtor, memory alloc and dealloc funcs.
- How should init'n be different from assignment?
    - They should be different as they correspond to different func calls.
- What does it mean for the obj to be passed by value?
    - Affects copy/move constructors.
- What are restrictions on legal values?
    - Only some combinations of values for a class's data members are valid
    - This imposes invariants of the class, which must be maintained
    - Affects ctors, assgmt oprts, setter funcs, exceptions to throw
- Does the new type fit into an inheritance graph?
    - If it inherits, your whole design is constrained by that class.
- What type conversions are allowed?
    - If want explicit only, write funcs to perform the conversions and use `explicit` on one param ctors
    - If want implicit, you need to write type conversion funcs/oprts
- What default funcs should be prohibited?
    - Use `= delete;` on ctors you don't want, etc.
- Who should have access?
    - Affects what are public, what are private, friends, and class nesting.
- What do this class guarantee / what's the invisible contract?
    - Performance, exception safety, resource usage, etc.
    - Imposes constraints on implementation.
- How general?
    - Affects what goes into the class template.
- Do I really need a new class?
    - If you just want to extend a class, perhaps use non-member functions or templates.

## 20 Prefer pass-by-ref-to-`const` to pass-by-value

Note: we aren't considering move semantics and other modern features here.

### ROT

- it's more efficient and avoids the slicing problem
- except built-in types, STL iterator and func objs

### reason: performance

Having a "naked" type in func param **causes the copy ctor to be called**, and subsequently, ctors of the members and superclasses. Also **dtors will be called** when the func returns.

Even some types might be small, **copying them can still be expensive**, e.g. copying an STL pointer copies everything it points to.

Even some types might be small and their copy operations cheap, **compilers may treat them differently** (e.g. placing a built-in `double` in a register while refusing to do so for a mere wrapper of `double`), leading to performance problems.

Pass-by-ref-to-const avoids creating new objects and thus calling ctors and dtors.

### reason: slicing

If you pass a derived class obj into a func that takes a base class obj, only the base class ctor is called and you're left with a "subset." This is particularly bad if you're using polymorphism: the base class version of the func is called.

### knowledge

- References are typically implemented as pointers, so they are cheap.
- You can pass-by-value **built-in types**, STL **iterator** and **func objs** because they're designed to behave when being copied. Aim for the same.

## 21 Don't return ref to local obj when returning an obj

Note: we aren't considering move semantics and other modern features here.

### ROT

- never return
    - ptr or ref to local stack obj
    - ptr or ref to static obj
    - ref to heap obj (ptr is very occasionally ok if it's managed by another env)

### reason

And, it doesn't bring any performance benifits. Consider:

- If a func wants to return a reference, that reference must point to an existing obj.
- In a function, an obj can either be on the stack or the heap.
- You either define a local obj or `new` one.
    - Local objects are allocated on the stack and will be destroyed when the func returns. If you return a reference to such object (which doesn't exist any more), you get UB.
    - Using `new` is very bad because users have to `delete` and sometimes there's no way to `delete` at all.
    - Either way, you call the ctor and must call the dtor.

## 22 Declare data members `private`

### reason aginst `public`

- **Consistency**: Client won't need to remember what's accessed through member variable or function.
- **Finer access control**: Public member is read-write access; using getter/setter can implement zero/read-only/read-write access (or ridiculously, write-only).
    - Many data members should be hidden. Not all needs a getter/setter.
- **Encapsulation**: If you have a computed property, using functions allow you to change underlying implementation without making clients to change their code.
    - You can ensure the class invariant is maintained because clients can't mess with them
- **Flexibility**: You can easily extend the getting/setting action with other things you need to do before and after (like a pre-commit hook in git)

### reason against `protected`

Think about encapsulatednessof a data member (inversely proportional to the amount of code breaking when it's changed). If it's public, when it changes, it breaks all client code that uses it. If it's protected, when it changes, it breaks all derived class code that uses it.

## 23 Prefer non-member non-friend funcs to member funcs

### ROT

- encapsulation, packaging flexibility, functional extensibility

### scenario

We have a func that simply calls other member funcs. Do we also make that a member func, or do we make it a non-member non-friend func accepting one param?

### encapsulation

Encapsulation is not solely about abstraction; its ultamate goal is to minimize exposure so it can be changed effortlessly. Making another member func adds to the number of "things" that can access the private members. If that is not necessary, don't make it a member.

Another benefit is that changing that outlying func doesn't require recompiling the whole class.

Note that the "member" we're talking about here is wrt the class the func can be written in. It can be the member of another class, which won't affect the encapsulation of the main class in concern.

### in practice

You can simply put the func in the same namespace as the class.

- It expresses the meaning naturally and signifies that it's a non-mem non-fren func (a conveniece func).
- Namespaces can span across multiple files, so if you have a lot of convenience funcs, you can put them in different headers for better modularity.
    - Note that you have many STL headers but all of them are in `std::`
- Clients can easily extend the set of conv funcs.

## 24 Declare non-member funcs when type conversions should apply to all params

This is especially true for numerical classes. If the operator a member function, the first param must be under this class, i.e. type conversion won't work on the first param.

If you don't need access to the internals, you don't need the operator to be `friend`.

## 25 Consider support for non-throwing `swap`

### background

You may want to implement your own `swap`. `std::swap` might be too heavy because it calls copy ctor 3 times. If you have a pimpl class, `std::swap` will still copy their underlying data, while actually only the pointers need to be swapped.

### `std::swap` with a member `swap`

We try to extend the `std::swap` with a *total template specialization* for when `T` is a `Widget`, but we can't directly access the underlying pimpl because it's private. Thus, we provide a `Widget::swap` to be called by the `std::swap<Widget>`. This is simple encapsulation.

```cpp
class Widget {
public:
	void swap(Widget& other) {
		using std::swap; // explained later [here](https://www.notion.so/nerrons/Effective-C-73e0037beb024637b5e5e08848f8d4fd#ba98de57cc66491684fc79fd3a705225)
		swap(pImpl, other.pImpl);
	}
};
namespace std {
	template<> // marks total template specialization
	void swap<Widget>(Widget& a, Widget& b)
	{ a.swap(b); }
}
```

### when it's a class template

If `Widget` is a class template that takes a typename `T`, the template specialization for `swap` fails (because funcs can't do *partial*). We just overload:

```cpp
template<typename T>
void swap(Widget<T>& a, Widget<T>& b) // No angle brackets
{ a.swap(b); }
```

But the `std` namespace is protected so you can't just *add* another function with a *new* signature. So we put that func in our custom namespace:

```cpp
namespace WidgetStuff {
	template<typename T> class Widget { /* ... */ }
	template<typename T> void swap(Widget<T>& a, Widget<T>& b)
	{ a.swap(b); }
}
```

Then if the user calls `swap` on two `Widget` objects, the *argument-dependent lookup* or *Koenig lookup* finds this `Widget`-specific version in `WidgetStuff`. Of course, even if your class is not a class template, you can also write a non-member swap like this.

### making it easy for the clients to call the correct version

Now we have three versions of `swap`, ranked from the least to the most desired:

- good o' `std::swap` (must exist)
- specialized `std::swap` (may or may not exist)
- `T`-specific `swap` (may or may not exist, and may or may not be in a namespace)

The client wants to simply call a `swap` and let the compiler design whether to call the more specific versions or fall back to the general `std::swap`.

To ensure this fall back behaviour, the client must have `std::swap` available when a `swap` is called. simply putting a `using std::swap;`before the `swap` call will suffice.

Don't qualify with an `std::` in front.

This is also why we need to follow the `using std::swap;` rule even inside the member `swap`: without it, it can only resolve to the `T`-specific `swap`, which might not be defined.

### recipe

(If default `swap` is acceptable, don't mess around like this. Else:)

- Offer public member `swap` func that efficiently swaps the value of two objects of your type (while, of course, maintaining the identity). Prevent emitting exceptions using `noexcept`.
- Offer non-member, same-namespace `swap` that calls member `swap`.
- If not class template, specialize `std::swap` for your class that calls member `swap`.

Finally, when you call `swap`, precede with `using std::swap;` and never qualify.

# Implementations

This chapter is about writing definitions of things. Do make declaration and design good though.

## 26 Postpone variable definitions as long as possible

- We shouldn't define unused variables. It's easy to avoid totally unused variables but not so for the "partially" unused in some branches.
- Also don't define a variable until you have a value to initialize it. This avoids calling the default ctor.
- Define variables insed the loop by default (clearer and doesn't pollute), unless assignment is cheaper than constr'n/destr'n.

## 27 Minimize casting

### ROT

- Casting harmful to the static typing system.
- If you have to, hide the cast in a function to save clients from casting themselves.
- Use new-style casts.

### new-style casts

- `const_cast`: rip off the constness of the object.
- `dynamic_cast`: perform "safe downcasting," i.e. to determine whether an object is of a particular type in an inheritance hierarchy.
    - cannot be performed using the old-style syntax.
    - significant runtime cost
- `reinterpret_cast`: for low-level casts that yield implementation-dependent results.
    - should be rare outside of low-level code
- `static_cast`: force implicit conversions or reverse such conversions.
    - non-`const` to `const`, `int` to `double`
    - `void*` to `T*`, `Base*` to `Derived*`

Prefer new-style casts at all time.

### type conversions modify values

Type conversions (implicit or explicit) usually are not only "treating bits differently," they often have runtime costs. Example:

```cpp
Derived d;
Base* pb = &d; // implicitly convert Derived* to Base*
```

The values of `&d` and `pb` are not necessarily the same. One object can have more than one addresses when being pointed to by different types of pointers!

Don't perform casts with wrong assumptions. Things can be different among compilers!

### `static_cast` might make a copy

One common problem is calling the base virtual func in the derived overriding func's impl'n. If you do a cast on `*this`, you actually create a base class copy instead of treating `*this` differently:

```cpp
class Base {
public:
	virtual void f() { /* Base impl'n... */ }
};
class Derived : public Base {
public:
	virtual void f() {
		// How do I call the base f() here?
	}
};
```

You might be tempted to write:

```cpp
/* !!! WRONG !!! */
static_cast<Base>(*this).f();
/* Derived-specific operations... */
```

But this will call the base `f()` on the casted `Base` copy and continue to modify `*this`, leaving `*this` in an invalid state. It should be:

```cpp
Base::f();
```

### refactoring `dynamic_cast`

First of all, it's **incredibly slow**.

One common use case of `dynamic_cast` is: you have a pointer- or reference-to-base, but you believe that pointed-to object is actually a derived object, and you want to perform operations with that assumption.

Two possible ways to solve that:

- Just use pointer- or reference-to-derived!
- Make a virtual func in the base class that by default does nothing, and override it with the operation you want to perform in the derived class. Just call the function!

Don't use cascading `dynamic_cast`s. It should be virtual funcs.

## 28 Avoid returning handles to object internals

A data member of an object is only as encapsulated as the most accessible function that returns a handle to it (references, pointers, iterators, etc.). Thus, returning a handle is risky.

### scenario

Let's say we have a `const` `Rect` that consists of two internal `Point`s, and we have functions that access them.

- If we return a reference to the `Point`, we immediately violate the constness of the `Rect`.
- If we return ref-to-`const` of the `Point`s, it can be bad in other ways, like *dangling handles*.

## 29 Strive for exception-safe code

It's not about catching everything or not emitting anything! It's about whether there's corruption.

### ROT

- no corruption when thrown
- try copy-and-swap
- write exception-safe code by default, unless it needs legacy

### requirements for exception safety

- **No leaked resources.**
- **No corrupted data structures.**

### three guarantees

- **Basic guarantee:**
    - When thrown, the program is still in a valid state, all invariants are still satisfied.
    - The exact state may not be predictable.
- **Strong guarantee:**
    - When thrown, the state of the program is unchanged.
    - Calls to such funcs are *atomic* because they either succeed completely or do nothing.
    - Easier to work with because only need to consider two possible outcomes
- **No-throw guarantee:**
    - The func always does what it's supposed to do.
    - All operations on built-in types are no-throw.

### fine-tuning statements to improve exception safety

- Don't mark something done (by counter, etc.) until it is really done and didn't throw.
- Use RAII objects to deal with ctor/dtor safety.

### copy-and-swap

**Principle:** Copy the object; make all needed changes on that copy; if anything goes wrong, the original copy remains unchanged; if all succeeds, swap the original and the copy.

**Practice:** Use the pimpl idiom (put all per-object data from the "real" object into a separate impl'n only object and give the real object a handle to that impl'n only object).

- You can make the impl'n object a struct because encaps'n should be done by the real object.

### not a panacea

**Feasibility:** Copy-and-swap for the object itself seems really promising because it guarantees the operations on the object itself is all-or-nothing, but it doesn't have power to control the side effects. When the function modifies non-local data, it's very hard to provide a strong guarantee.

**Efficiency:** Copying an object is not negligible task.

### insights

- Don't be too hard on yourself for not offering strong guarantee.
- If a func is not exception-safe, any funcs that build upon it will not be exception-safe.
- If a single func in the whole system is not exception-safe, the whole system is not.

## 30 Understand inlining

### ROT

- inline code can increase size of program since it's repetitive
- limit most inlining to small, frequently called funcs
- think debugging, binary upgradability, speed
- don't declare func templates `inline` just because they're all in header files

### declaring an inline func

- **Impicitly**: Just write down the func body inside the class def
- **Explicitly**: In the impl'n file, put an `inline` keyword in front of func def

### inline and template

They are independent of each other.

- You can declare a func template `inline` if all funcs from the template should be inlined.
- If you don't want the funcs from the template to be inlined, avoid declaring so (exp and imp).

### compilers and inlines

- Compilers are free to refuse inlining a func
- Even if compiler is willing to inline a func, it might or might not actually get inlined due to different ways of calling it (direct vs. through func ptr)
- If you distribute binaries of your func...
    - ...and it's inlined, when you update the func and the binary, your clients must recompile everything that uses that func.
    - ...and it's not inlined, the client only needs to relink.
- Debuggers often have troubles with inline funcs. Many just disable inlining for debug builds

## 31 Minimize compilation dependencies between files

### problem

When you write a library, you have an interface and corresponding impl'n. When you change something the impl'n, you don't want the client to recompile everything related to that "something." You only want to allow recompilation when the interface changes.

### dependency chain

If you `#include` a header that has classes that your class `T`  needs, when that header changes, the file containing `T` needs to be recompiled. Worse, this process is recursive, thus a chain.

You don't want recomp'n on impl'n change, you need to break the chain using forward declaration. 

C++ doesn’t do a very good job of separating interfaces from implementations. You have to manually simulate an interface: a promise to the compiler that something can be used.

### essence

The essence: replace dependencies of definitions with dependencies on declarations.

To achieve such separation, make your header files self-sufficient when practical, and when it's not, depend on declarations in other files, not definitions.

### handle class (pimpl idiom)

- Put all impl'n details into an impl'n class; `#include` it in your Handle class's file; forward all calls to your Handle class to the impl'n class. This way you don't even break clients' code.
- The goal of a Handle class is to break the chain between Handle class and Impl'n class.
    - What if we try to break the dependency chain directly, by forward-declaring all the dependencies of your good o' monolithic class?
        - There will be too many things that can't or is very difficult to be forward-declared.
    - If we use the pimpl idiom, in the header file of the Handle class, the Impl'n class is easily and cleanly forward-declared.
        - For example, inside the Impl'n class, you can `#include` whatever you like.
- But we still have to provide an impl'n for the Handle class itself, after all.
    - In the `.cpp` file, it's true that we must include both headers. But remember it's different from using `#include` in the `.h` file, like the monolithic class does!
    - Just init the ptr to the Impl'n class forward all member funcs to it as well.
- Cost:
    - each object has size increased by the ptr to the Impl'n class
    - each call to the member functions needs to go through one redirection
    - the Impl'n class is allocated on heap so all problems with dynamic allocation is present
    - can't freely use inline functions

### advice regarding handle class

- Avoid directly using objects when using handles will do.
- Depend on class declarations instead of class definitions when possible.
    - See the [Notes](https://www.notion.so/nerrons/C-Random-bits-e4aed87017ab4640b37863403d30df52#e879bd29355c403a92ad3d389f198657).
- Provide separate header files for declarations and definitions if necessary.
    - You'll have to invest in keeping them consistent.

### interface class

- An Interface class typically contains no data members, no ctors, a virtual dtor and a set of pure virtual member functions.
    - The client always uses the Interface class and should not be aware of the concrete class
- Using an Interface class "reshapes" the dependency chain: the client and the concrete class both depend on the interface class, so whether the concrete class changed won't affect client.
- Since clients can't call ctors of concrete funcs, the Interface class should provide a virtual member function that acts as a ctor.
    - These are called factory functions and are usually static inside the Interface class
    - Insane! You can let `D` inherit `B` while a function in `B` mentions `D`
- Cost:
    - every function call is virtual
    - objects derived from the Interface class contains a virtual table pointer
    - can't freely use inline functions

Interface classes are less transparent then Handle classes because clients must use ptrs or refs to access the class (because it's virtual)

# Inheritance and OOP

Decisions to make:

- inheritance, single or multiple?
- inheritance link, private, protected or public? virtual or non-virtual?
- member function options, virtual, non-virtual or pure virtual?
- other language features (default params, lookup rules, design options, etc)

Tools provided by C++ have meanings; they are not just syntax to be followed, they express what you want to say about your software.

## 32 Make sure public inheritance models "is-a"

Public inheritance means "is-a."

### semantics

- If class `D` inherits from class `B`, every object of type `D` is also of type `B`, but not vice versa.
    - `B` is a more general concept than `D` and `D` is a more specialized concept than `B`
    - Anywhere a `B` object can be used, a `D` object can be used, precisely because a `D` object is a `B` object.
    - Anywhere a `D` object can be used, a `B` object does not suffice, presisely because a `B` object is not a `D` object.
- The model in code does not necessarily reflect how we talk about "is-a" in daily language.
    - E.g. we say a penguin is a bird and we say a bird can fly. If we make `Penguin` publicly inherit from `Bird` then a penguin must support flying, which is wrong.
        - You shouldn't override the `fly()` method of `Penguin` and make it throw, because calling `Penguin::fly()` will compile, and it's semantically wrong.

### caution

- If `D` has more strict class invariants by `B`, the methods from `B` might break `D`.

## 33 Avoid hiding inherited names

Naturally, names in `D`'s scope will shadow the same names in `B`'s scope.

### hidden overloads

If `B` has virtual member function overloads that is not defined in `D` (e.g. when `f(double)` and `f(string)` is defined in `B` but only `f(double)` is defined in `D`), all those overloads are hidden, so you can't call base class overloads from derived object.

This violates the "is-a" principle because now `D` is not a `B` any more (cuz it doesn't support all functions of `B` that the client might use, which will include those overloads)

### solution for public inheritance

Explicitly make base class overloads visible, by adding `using B::f` after the `public:` of `D`'s definition. This is compulsory [cuz it's a violation otherwise](https://www.notion.so/nerrons/Effective-C-73e0037beb024637b5e5e08848f8d4fd#0162aba80c7546798f8f719a9270e978).

### solution for private inheritance

You don't have to support everything provided by `B`; you can manually pick what overloads `D` should support and forward them to `B` with an inline function.

## 34 Differentiate between inheritance of interface and inheritance of implementation

This item concerns inheritance of functions.

C++ inheritance is convoluted. There are three kinds of behaviours for func inht'ce:

- Inherit only the interface (declaration)
- Inherit both interface and impl'n, and allow overriding
- Inherit both interface and impl'n, but disallow overriding

The interfaces are always inherited by public inheritance.

### pure virtual function

Have derived classes inherit a function *interface only*. All derived classes must supply this function but impl'n is not enforced in any way.

You **can** supply a definition for a pure virtual function, but the only way to call it is through qualification in derived classes.

### impure virtual function

Have derived classes inherit a function *interface and a default impl'n*.

This might be dangerous cuz derived class can accidentally inherit the default behaviour even when it's not explicitly specified to do so. If you want things to be on the table, you need to make the function in the base class pure virtual, and either:

- Provide a protected member function that acts as a default impl'n that can be used by any derived class;
    - You can give the default impl'n a `protected` level at the cost of using another name;
- Or, provide a definition for that pure virtual function.
    - You can save the extra name but the impl'n has to be `public`. But hey, if you were still using an impure virtual function, it's not gonna be better.

### non-virtual function

Have derived classes inherit a function *interface and a mandetory impl'n*.

### common mistakes

- Mistake #1: declare all functions non-virtual in classes that can act as base classes.
    - No room for specialization; Non-virtual destructors might just be wrong
    - Arguments on performance: the 80/20 rules says that 80% of runtime is spent on 20% of code. Compared to worrying about virtual calls please do profiling and real optimization work if speed is desired.
- Mistake #2: declare all function virtual in classes that don't act as interfaces.
    - It can be a sign of a class designer who lack the backbone to take a stand.
        - Not everything should be redefinable.
        - If so, the class has no invariants, questioning the necessity of its very existence.

## 35 Consider alternatives to virtual functions

Say your `B` has a public virtual member function `B::vf()`.

### ROT

- alternatives include NVI and Strategy
- moving member functions outside brings the problem of accessing non-public members

### *template method pattern* via *non-virtual interface* idiom

Make `vf()` private and create an `f()`, the *wrapper*, that calls `vf()`.

- Advantage: able to do before/after stuff in the wrapper, which will be inherited by all `D`s.
- It's OK to redefine private member functions because *when* and *how* are two things.
- Sometimes `B::vf()` must be protected, like when `D::vf()` needs to call it.

### *strategy pattern* via function pointers

Instead of providing an interface in the class, declare a data member to be a function pointer and pass it in during constr'n.

- Advantage: Different instances of `D` can have different funcs.
- Advantage: The funcs can be changed at runtime.
- Disadvantage: Might weaken encapsulation if the func needs to access internal members.
    - declare the func as `friend`, make more params in the func, expose more public...

### *strategy pattern* via `function`

Like above but a `function` holds any callable entity and takes care of conversion.

### *classic strategy pattern*

Separate the hierarchies. So you want to customize the behaviour of your function in your derived classes... You can turn that function into a class `F`, and make `F` a member of `B`. 

- `F` needs a virtual function to be overridden by its derivatives.
- `B` can take derived classes of `F` and call that virtual func. This is a form of polymorphism.

## 36 Never redefine an inherited non-virtual function

Otherwise, how the object behaviour has nothing to do with the object itself but with the type of the pointer to it:

```cpp
D x;
B* pB = &x;
D* pD = &x;
pB->f(); // calls B::f()
pD->f(); // calls D::f()
```

It's also semantically incorrect, because declaring a base member function non-virtual means "this function interface should be preserved in all classes that can be treated `B`."

## 37 Never redefine a func's inherited default param value.

First of all, you shouldn't redefine non-virtual functions in any way. Let's limit to virtuals.

### static & dynamic types

- *static type*: the type you declare in the program text.
- *dynamic type*: the type of the object to which the name currently refers.
    - not the pointed-to object; `D x; B* pB;`, `pB` has the dynamic type `D*`, not `D`

### the keng

Default param values are **statically bound**, not dynamically bound; meaning, they are determined by the object's static type. This is for runtime efficiency.

You also can't leave out the default param in func override of derived class because it won't get inherited.

### solutions

Use NVI and put the default param value only to the wrapper.

## 38 Model "has-a" & "impl'ed-in-terms-of" via composition.

### terminology

*Composition* arises when objects of one type contain objects of another type.

There are two *domains* in your software:

- *application domain* (corresponds to "has-a")
- *impl'n domain* (corresponds to "impl'ed-in-terms-of")

### diff between "is-a" & "impl'ed-in-terms-of"

This determines whether you should use public inheritance.

E.g. you want to impl't a `Set` using `std::list`. You shouldn't publicly inherit `list` because they have different behaviours (when adding duplicate items). You should use composition.

`Set` "is-not-a" `list`. `Set` is "impl'ed-in-terms-of" a `list`.

## 39 Use private inheritance judiciously

### ROT

- private inheritance means "impl'ed-in-terms-of"
- used when a derived class needs
    - access to protected base class members
    - to redefine inherited virtual functions
- empty base optimization

### behaviour of private inheritance

- Compilers will not implicitly convert derived class to base class
- In the derived class, inherited members are private, regardless of their visibility in base class

### meaning

- Means "impl'ed-in-terms-of."
- You privately inherit `D` from `B` because you want to take advantage of some features available in `B`.
- Private inheritance is purely a impl'n technique. It's not about designing.
- Referring to [Item 34](https://www.notion.so/nerrons/Effective-C-73e0037beb024637b5e5e08848f8d4fd#60e1513cc4e4441ab8a6a7b13a439127), private inheritance only inherits impl'n, not interface.

### when to use

While they express the same meaning, almost always use composition because it's simpler. There are cases when you must use private inheritance:

- You want to override a virtual function in the integrated class.
- You want to access a protected member of the integrated class.

### example: integrating a utility class

You have a `Timer` class:

```cpp
class Timer {
public:
	explicit Timer(int tickFreq);
	virtual void onTick() const;
};
```

You integrate that in your `Widget` using private inheritance:

```cpp
class Widget: private Timer {
private:
	void onTick() const override; // Redefine the virtual function
};
```

So you can reuse things in `Timer` without exposing `onTick`.

But private inheritance is not the only way. You can use composition if you like, by wrapping a "inheritor" class that inherits `Timer` to use its virtual interface:

```cpp
class Widget {
private:
	class WidgetTimer : public Timer {
	public:
		virtual void onTick() const;
	};
	WidgetTimer timer;
	// ...
};
```

This has two advantages:

- You can prevent derived classes from `Widget` to redefine `onTick`
    - If `Widget` privately inherits from `Timer`, `Widget`'s subcls can redefine `onTick`
    - Since `WidgetTimer` is private, `Widget`'s subcls don't have access to it
- It eliminates a compilation dependency
    - If `Widget` privately inherits from `Timer`, `Timer`'s definition must be available when `Widget` is compiled, leading to an `#include "Timer.h"`.
    - Move `WidgetTimer` outside and use a pointer to it in `Widget`➡️forward-declaration.

### edge case: *empty base optimization*

This only concerns privately inheriting from an empty class to reduce size...

## 40 Use multiple inheritance judiciously

### problems with MI

- Ambiguity: it's possible to inherit the same name from more than one base class
    - Compiler refuses to compile when ambiguity is found
    - Qualify the call with base class namespace
- Deadly MI Diamond:
    - Happens when `D` inherits from `B1` and `B2`, which both inherit from `A`.
    - C++ will replicate all data member fields inherited from `A`.
    - To override the default behaviour:
        - Make the class with the duplicatable data a *virtual base class*. This way make references "resolve" to the correct instance and leave only one piece of data.
        - It's most practical when the base class has no data

### One possible use...

Multiple inheritance is very useful for inheriting a publicly inheriting an interface (possibly pure virtual) class and privately inheriting a class that facilitates impl'n.

# Templates and Generic Programming

## 41 Know implicit interfaces & compile time polymorphism

### explicit vs implicit; compile time vs runtime

Without templates...

- When you declare an object to be of a type `Widget`, that object must support an *explicit interface*, i.e. the interface defined by `Widget`.
- Virtual functions exhibit *runtime polymorphism*: the specific function to call is determined at runtime based on that object's dynamic type.

With templates...

- The interface the object must support can no longer be named easily (because no idea what exactly the type is), and will depend on the series of operations performed on that object.
- Operations on that object may involve instantiating templates, which occurs during compilation. We have *compile time polymorphism* because **i**nstantiating function templates with **different template params** leads to **different functions being called**.

### more on explicit and implicit interfaces

- An explicit interface is usually a set of function signatures/declarations.
- An implicit interface is a sequence of valid **expressions**.
    - For operators, since they have overloading, an `a op b` expression is valid if there exists an operator `op` that takes types `X` and `Y`, where `a` has type `A` which can be implicitly converted to `X`, and `b` has type `B` which can be implicitly converted to `Y`.
- Both are checked during compilation. Violation causes the code fail to compile.

## 42 Understand the two meanings of `typename`

- `template<class T>` == `template<typename T>`

### asserting nested dependent type names

Things like `T::const_iterator` are

- *dependent names*: Names in a template that are dependent on a template parameter.
- *nested names*: Names nested inside a class.

C++ consider nested dependent type names ambiguous by default; so we need to prepend a `typename` to the name.

E.g. `C::type`. We don't know what `C` is so we don't know if `type` is really a type.

Only exceptions where you should not prefix `typename` when using nested dependent names:

- in the list of base classes when declaring a derived class
- in member initialization lists

## 43 Know how to access names in templatized base classes

Say we have this, which won't compile:

```cpp
template<typename T>
class D : public B<T> {
public:
	void fd() { fb(); } // fb() is supposed to be some function from B<T>
};
```

### reason

Prior to template instantiation, the compiler doesn't know what exactly the base class is, and consequently doesn't know what names it has.

This is not a limitation of the compiler, and is reasonable behaviour. You can actually make the name missing in one of the `B<T>`s using partial/total specialization:

```cpp
template<>
class B<int> {}; // we don't have fb() at all when T == int!
```

### solutions

If you're sure `fb()` exists in all instantiations of `B<T>`, you can force the compiler to search `B`. Note that if this assumption is found not true during instantiation, a compilation error is thrown.

- Prepend with `this->`.

    ```cpp
    void fd() { this->fb(); }
    ```

- Employ a `using` declaration (See [Item 33](https://www.notion.so/nerrons/Effective-C-73e0037beb024637b5e5e08848f8d4fd#615a8ccb951843248fa2c29d9e333d6e)).

    ```cpp
    template<typename T>
    class D : public B<T> {
    public:
    	using B<T>::fb;
    	void fd() { fb(); }
    }
    ```

- Explicitly qualify the function call of `fb()`. This is the least desirable: if `fb` is virtual, explicit qualifications turn off the virtual finding behavior.

    ```cpp
    void fd() { B<T>::fb(); }
    ```

## 44 Factor parameter-independent code out of templates.

### ROT

- Make a new base class that has less template params.
- Put functions that don't absolutely need a template param there.
- Take care of communication: usually the derived needs to expose something to the base.

### terminology

- *code bloat*: **binaries with (almost) replicated code.
- *commonality and variability analysis*: look at the code (function/class/template), extract what's common, and customize the common stuff with the varying parts.

### implicit replication about templates

In non-template code, repetition is explicit in the source code. But for template code there's only one copy of the source code and repetition occurs only after instantiation.

### scenario

Say we have a class representing a square matrix, with the dimension inside the type param:

```cpp
template<typename T, std::size_t n>
class SquareMatrix {
public:
	void invert();
};
```

Then if we instantiate a lot of different square matrices, we will be instantiating as much `invert()` function too! While we maintain the interface of `SquareMatrix` which has two template params, we can internally put the `invert()` to somewhere else, namely a base class.

- Code:

    ```cpp
    template <typename T>
    class SMBase {
    protected: // invert is intended to be used by only SM
    	void invert(std::size_t dim); // This shouldn't be inlined
    	// ...
    };

    template<typename T, std::size_t n>
    class SM : private SMBase<T> {
    private:
    	using SMBase<T>::invert;
    public:
    	void invert() { invert(n); } // no additional cost cuz implicit inline
    	// ...
    };
    ```

However, this way `SMBase` has no idea where is the actual numerical data for the matrix. To avoid passing a pointer in all functions including `invert`, we give `SMBase` data members.

- Code:

    ```cpp
    template<typename T>
    class SMBase {
    protected:
    	SMBase(std::size_t n, T* pMem) : size(n), pData(pMem) {}
    	void setDataPtr(T* ptr) { pData = ptr; }
    	// ...
    private:
    	std::size_t size;
    	T* pData;
    };

    typename<typename T>
    class SM : private SMBase<T> {
    public:
    	SM() : SMBase<T>(n, data) {}
    private:
    	T data[n*n];
    };
    ```

If the data is large, we can put it on the heap.

- Code:

    ```cpp
    template<typename T, std::size_t n>
    class SM : private SMBase<T> {
    public:
    	SM :
    		SMBase<T>(n, nullptr),
    		pData(new T[n*n])
    	{ this->setDataPtr(pData.get()); }
    private:
    	std::unique_ptr<T[]> pData;
    };
    ```

### conclusion

**Final results:** 

- Many of `SM`'s member functions can be simple inline calls to (non-inline) base class versions that doesn't get replicated when dealing with matrices of different dimensions.
- Meanwhile, `SM<int, 5>` and `SM<int, 10>` are still different types.
- Decreasing executable size can improve locality of reference in instruction cache.

**Potential costs:**

- Hardcoded versions can generate more optimized instructions.
- You need complex mechanisms for the functions in the base class to use data from the objects in the derived class.
    - In our case, we use pointers. But there can be tons of decisions to make.
    - Can increase object size.
    - Can impose resource management difficulties.

Also, not only non-type template params lead to bloat. Like `vector<long>` and `vector<int>`, or different types of pointers, might be identical.

## 45 Use memfunc templates to accept "all compatible types"

### ROT

- use member function templates to accepts compatible types
    - use other internals to constrain
- declare normal copy ctor and copy asgn oprt along side generalized ones

### background

We may write *smart pointers*: objects that act like pointers but add functionalities. We want the smart pointers to support implicit conversions, like so:

```cpp
SP<Top> pt1 = SP<Middle>(new Middle);
SP<Top> pt2 = SP<Bottom>(new Bottom);
SP<const Top> pct = pt1;
```

By default they are considered totally different classes.

### appetizer: *general copy ctors*

To support infinitely many conversions, we need ctor templates, or, *generalized copy ctors*.

```cpp
template<typename T>
class SP {
public:
	template<typename U> // not explicit so it supports implicit conversion
		SP(const SP<U>& other);
};
```

But that offers definitely too much possibilities (like you can construct an `SP<Bottom>` from an `SP<Top>`. To constrain the conversions, rely on the behavior of built-in pointers:

```cpp
public:
	template<typename U>
		SP(const SP<U>& other) : heldPtr(other.get()) { /* ... */ }
	T* get() const { return heldPtr; }
private:
	T* heldPtr;
```

This only compiles if there's an implicit conversion from `U*` to `T*`.

### supporting assignments and other conversions

- A possible declaration of `shared_ptr`.

    ```cpp
    template<typename T>
    class shared_ptr {
    public:
    	template<typename Y>
    		explicit shared_ptr(Y* p);
    	template<typename Y>
    		shared_ptr(shared_ptr<Y> const& sp);
    	template<typename Y>
    		explicit shared_ptr(weak_ptr<Y> const& wp);
    	template<typename Y>
    		explicit shared_ptr(unique_ptr<Y> const& up);
    	template<typename Y>
    		shared_ptr& operator=(shared_ptr<Y> const& sp);
    	template<typename Y>
    		shared_ptr& operator=(unique_ptr<Y> const& up);
    };
    ```

From the example above,

- All ctors except the generalized copy constructor are `explicit`.
    - implicit conv'n from `shared_ptr<X>` to `shared_ptr<Y>` is allowed
    - implicit conv'n from a raw pointer or another kind of smart pointer is not allowed
    - explicit conv'n of above is fine

### about the default ctors/operators

Declaring a generalized copy ctor doesn't stop the compiler from generating a "normal" ctor.

- To take full control, declare both of them:

    ```cpp
    template<typename T>
    class shared_ptr {
    public:
    	shared_ptr(shared_ptr const& sp);
    	template<typename Y>
    		shared_ptr(shared_ptr<Y> const& sp);
    	shared_ptr& operator=(shared_ptr const& sp);
    	template<typename Y>
    		shared_ptr& operator=(shared_ptr<Y> const& sp);
    };
    ```

## 46 Define non-member functions inside templates when type conversions are desired

### ROT

Implicit conversions are never considered during template argument deduction.

When writing a class template that offers functions related to the template that should support **implicit conversions** on **all parameters**, define those funcs as friends inside the class template.

### problem: function templates doesn't have implicit conversions

Suppose we have a `Rational` class with a `*` operator. (See [Item 24](https://www.notion.so/nerrons/Effective-C-73e0037beb024637b5e5e08848f8d4fd#f6865af88753461c93532e6dc93ce666) for oprt declaration)

```cpp
template<typename T> class Rational {
public:
	Rational(const T& numerator = 0, const T& denominator = 1);
	const T numerator() const;
	const T denominator() const;
};

template<typename T>
const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs)
{ /* ... */ }
```

Even though the oprt allows implicit conversions in both of their operands, this won't compile:

```cpp
Rational<int> oneHalf(1, 2); // OK
Rational<int> one = oneHalf * 2; // ERROR! Won't compile
```

Because the compiler doesn't know the type of the second operand.

- Deduction about `oneHalf`: `*`'s first param is previously declared `Rational<int>`, so the `T` must be `int`.
- Deduction about `2`: `Rational` has a non-explicit ctor that will accept a `2`, but it won't happen during template argument deduction (like in the callout above).
    - Such implicit conv'ns happen during function calls, but not during "deciding what functions are valid" phase, which happens when instantiating a func template.

### solution: `friend` inside class template

A friend declaration in a template class can refer to a specific function.

We abuse that fact to declare a concrete function (not a func template) when instantiating a class, so we type deduction happens at function-call level instead of template-instantiation level, which utilizes implicit type conversion.

```cpp
template<typename T> class Rational {
public:
	friend const Rational operator*(const Rational& lhs, const Rational& rhs);
	/* Since in class template X, X means X<T>, that is a shorthand for this:
	friend const Rational<T> operator*(const Rational<T>& lhs,
																		 const Rational<T>& rhs); */
};
```

Then as soon as we declare a `Rational<int>`, the class `Rational<int>` is instantiated, which automatically declares the friend function `operator*` which takes `Rational<int>` params.

That solved, we also need to define the operator somewhere. Maybe just in-place:

```cpp
template<typename T> class Rational {
public:
	friend const Rational operator*(const Rational& lhs, const Rational& rhs); {
		return Rational(lhs.numerator() * rhs.numerator(), /* ... */);
	}
};
```

Notice that the friend function doesn't access anything that's not public. Why `friend`?

- To make implicit conversions available on all parameters, we need to declare the operator **a non-member function**. (See [Item 24](https://www.notion.so/nerrons/Effective-C-73e0037beb024637b5e5e08848f8d4fd#f6865af88753461c93532e6dc93ce666)).
- To make implicit conversions available for function templates, we need to declare the function **inside the class**, so a concrete function gets instantiated with the class.

The only way to do that is with `friend`.

### improvement: minimize inlining

The solution above will lead to inlining the operator function. 

- To override that behavior, use the "friend calls helper" approach:

    ```cpp
    template<typename T> class Rational;

    template<typename T>
    const Rational<T> doMult(const Rational<T>& lhs, const Rational<T>& rhs);

    template<typename T> class Rational {
    public:
    	friend const Rational operator*(const Rational& lhs, const Rational& rhs)
    	{ return doMult(lhs, rhs); }
    };

    template<typename T>
    const Rational<T> doMult(const Rational<T>& lhs, const Rational<T>& rhs) {
    	return Rational(lhs.numerator() * rhs.numerator(), /* ... */);
    }
    ```

The `doMult` will not support mixed-mode multiplication, but it doesn't need to, because the operator will invoke the correct `doMult` specialization.

## 47 Use traits classes for information about types

This section will need an update for C++11 `using`.

### mini review: 5 types of iterators

- *input iterator*: move only forward, 1 step at a time, only read what they point to, only once.
    - modeling the read pointer into an input file (`istream_iterator`)
- *output iterator*: move only forward, 1 step at a time, only write to what they point to, once.
    - modeling the write pointer into an output file (`ostream_iterator`)
- *forward iterator*: plus read/write more than once (`slist`)
- *bidirectional iterator*: like forward iterators but can move back and forth
    - (`std::list`, `set`, `map`, `multiset`, `multimap`)
- *random access iterator*: perform iterator arithmetic (jump arbitrary distance in constant time)
    - modeling on built-in pointers (`vector`, `deque`, `string`)

### iterator tag

- For every kind of iterator there's a "tag struct."

    ```cpp
    struct input_iterator_tag {};
    struct output_iterator_tag {};
    struct forward_iterator_tag: public input_iterator_tag {};
    struct bidirectional_iterator_tag: public forward_iterator_tag {};
    struct random_access_iterator_tag: public bidirectional_iterator_tag {};
    ```

### introducing traits

*Traits* allow clients to get information about a **type** during **compilation**. They aren't a language feature; just a convention.

One requirement of using traits is that, traits should be applicable to built-in types as well; in other words, traits need to be a universal tool for all types. (e.g. when a built-in type doesn't have a trait, the program should be aware of that, instead of just break.)

Thus, traits are something **outside** of classes (because otherwise the built-in types won't have them). The standard way of implementing traits is to make a template of the trait `struct`(ironically called *traits classes*) and let different specializations have different content.

Traits are implemented differently for built-in types and user-defined classes.

### traits and user-defined classes

Simple! Let the class declare a trait for themselves and our trait class just gathers the info.

Provide a public `typedef` from the iterator tag to `iterator_category` (just a convension).

```cpp
template<typename T> class deque {
public:
	class iterator {
	public:
		typedef random_access_iterator_tag iterator_category;
	};
};
```

In the trait class, respect whatever the iterator says.

```cpp
template<typename IterT> struct iterator_traits {
	typedef typename IterT::iterator_category iterator_category;
};
```

### traits for built-in types

For iterators, we also need to consider raw pointers. Since we can't attatch trait information inside a "pointer class," we place the info under our trait class, using partial specialization.

```cpp
template<typename T> struct iterator_traits<T*> { // partial
	typedef random_access_access_iterator_tag iterator_category;
};
```

### procedure of designing and implementing traits

- Identify the info you need about types.
- Choose a name for that info.
- Provide a template and a set of specializations.

### using traits

Now consider `if constexpr`.

- You shouldn't just compare type traits in `if` statements cuz: it's runtime; it bloats exe size; it leads to compile problems cuz there can be mismatched types that still need to be compiled.

    ```cpp
    template<typename IterT, typename DistT>
    void advance(IterT& iter, DistT d) {
    	if (typeid(typename std::iterator_traits<IterT>::iterator_category) ==
    			typeid(std::random_access_iterator_tag)) {
    		iter += d; // Even when sometimes IterT doesn't support operator+=,
    							 // the compiler still requires this typecheck to pass.
    							 // That prevents compiling.
    	} else {
    		/* Do things without using operator+= ... */
    	}
    }

    std::list<int>::iterator iter; // Doesn't support operator+=
    advance(iter, 10); // ERROR!
    ```

You want a conditional construct that gets evaluated during compilation. That's overloading!

Create multiple aux functions for the function where you want to utilize traits, and use an additional parameter (called the *traits parameter*) to distinguish different traits.

## 48 Be aware of template metaprogramming

### Hello World: factorial

```cpp
template<unsigned n>
struct Factorial {
	// enum hack; every instantiation has a unique value
	enum { value = n * Factorial<n-1>::value };
};

template<>
struct Factorial<0> {
	enum { value = 1 };
};
```

### why even bother

- Ensure dimensional unit correctness.
- Optimize matrix operations.
    - e.g. when multiplying huge matices.
    - use *expression templates* to avoid temporary object creation and merge loops
- Generating custom design pattern implementations.
    - use *policy-based design* to create independent design choices.

# Customizing `new` and `delete`

## 49 Understand the behavior of the new-handler

### ROT

- use new-handlers to allocate more memory in case allocation fails
- be aware of the reusable way to make class-specific new-handlers using `set_new_handler`

### introducing new-handlers

Before `operator new` throws an exception due to memory allocation error, it calls the *new-handler* to handle it. To customize the behavior, use `std::set_new_handler` from `<new>`.

```cpp
namespace std {
	// new_handler = ptr to a func that takes and returns nothing
	typedef void (*new_handler)();

	// takes and returns a new_handler
	// throw() means the func shouldn't throw any exceptions
	// returns the old new_handler
	new_handler set_new_handler(new_handler p) throw();
}
```

### writing new-handlers

`operator new` calls the new-handler **repeatedly** if it can't allocate the memory. Therefore, a new-handler must do one of the following in one round:

- Make more memory available for the next round.
    - Popular impl'n: reserve large memory at start-up; release some for new-handlers.
- Install a different new-handler if needed.
    - If current new-handler can't make any more memory available, call `set_new_handler` with another one, or modify its own behavior.
- Deinstall the new-handler to force throwing.
    - Pass a `nullptr` to `set_new_handler`. Next round throws.
- Throw an exception.
    - Usually `bad_alloc` or derived. Propagates to site originating the request for memory.
- Not return.
    - Usually by calling `abort` or `exit`.

### proposal for class specific new-handlers

1. Put a static new-handler data member (`currentHandler`) in the class.
2. In the impl'n file, define the `currentHandler` to be `nullptr`.

Then, `Widget::set_new_handler` should play the same role as the global `set_new_handler`.

`Widget`'s `operator new` will do the following:

1. Call the global `set_new_handler` with `Widget`'s own new-handler.
2. Call the global `operator new` to perform actual memory allocation.
    1. If allocation fails, the global `operator new` invokes `Widget`'s new-handler.
    2. If ultimately cannot alloc memory, throw `bad_alloc`.
    3. `Widegt`'s `operator new` restores old global new-handler; propagate exception.
        - To ensure the old one is always restored, use resource-managing objs.
3. When the global `operator new` succeeds in allocation, `Widget`'s `operator new` returns a pointer to the allocated memory.
    - dtor of the obj managing global new-handler automatically restores the old one.

### implementing class specific new-handlers

- Make `Widget` class.

    ```cpp
    class Widget {
    public:
    	static std::new_handler set_new_handler(std::new_handler p) throw();
    	static void* operator new(std::size_t size) throw(std::bad_alloc);
    private:
    	static std::new_handler currentHandler;
    };
    ```

- Make `Widget`'s `set_new_handler`.

    It really isn't a `set_new_handler` cuz it doesn't change the new-handler at all; it just records the `Widget`-specific new-handler to be installed when `new`ing a `Widget`.

    Like the original `set_new_handler` it returns the old one so it can be restored later.

    ```cpp
    std::new_handler Widget::set_new_handler(std::new_handler p) throw() {
    	std::new_handler oldHandler = currentHandler;
    	currentHandler = p;
    	return oldHandler;
    }
    ```

- Make a resource managing class for global new-handlers.

    This holder holds one new-handler. Whenever this holder is destroyed, install the new-handler it holds.

    ```cpp
    class NewHandlerHolder {
    public:
    	explicit NewHandlerHolder(std::new_handler nh)
    	: handler(nh) {} // keep track of the new new-handler

    	~NewHandlerHolder()
    	{ std::set_new_handler(handler); }

    private:
    	std::new_handler handler;

    	NewHandlerHolder(const NewHandlerHolder&) = delete;
    	NewHandlerHolder& operator=(const NewHandlerHolder) = delete;
    };
    ```

- Override the `Widget`'s `operator new`.

    Use `NewHandlerHolder` to ensure new-handler restoration.

    ```cpp
    void* Widget::operator new(std::size_t size) throw(std::bad_alloc) {
    	NewHandlerHolder h(std::set_new_handler(currentHandler));
    	return ::operator new(size);
    }
    ```

- Client uses `Widget`'s new-handling just like the vanilla one.

    ```cpp
    void handmaster();

    Widget::set_new_handler(handmaster);
    Widget* pw1 = new Widget; // will call handmaster if alloc fails
    std::string* ps = new std::string; // will call old global nh if fails

    Widget::set_new_handler(nullptr);
    Widget* pw2 = new Widget; // throws immed. if alloc fails
    ```

### reusing the class-specific new-handler feature

We extract `Widget`'s `set_new_handler` and `operator new` to a "mixin-style" base class.

Then we turn it into a template, otherwise since those goodies are static, all classes that inherited from it will have the same copy.

- This is our `NewHandlerSupport` class.

    ```cpp
    template<typename T>
    class NewHandlerSupport {
    public:
    	static std::new_handler set_new_handler(std::new_handler p) throw();
    	static void* operator new(std::size_t size) throw(std::bad_alloc);
    private:
    	static std::new_handler currentHandler;
    };

    template<typename T>
    std::new_handler
    NewHandlerSupport<T>::set_new_handler(std::new_handler p) throw()
    {
    	std::new_handler oldHandler = currentHandler;
    	currentHandler = p;
    	return oldHandler;
    }

    template<typename T>
    void*
    NewHandlerSupport<T>::operator new(std::size_t size) throw(std::bad_alloc)
    {
    	NewHandlerHolder h(std::set_new_handler(currentHandler);
    	return ::operator new(size);
    }

    // init each currentHandler to nullptr
    template<typename T>
    std::new_handler NewHandlerSupport<T>::currentHandler = nullptr;
    ```

- This is how we use it.

    ```cpp
    class Widget : public NewHandlerSupport<Widget> {
    	/* as before, but without set_new_handler and operator new */
    };
    ```

Note that `T` inherits from `Something<T>`. It's called *curiously recurring template pattern* (CRTP). You can think of at as "Do It For Me": "I'm `T`, now I wanna inherit from the `Something` that's for me, and for me only."

### legacy nothrow operator new

```cpp
Widget *pw1 = new Widget; // throws bad_alloc if alloc fails
if (pw1 == 0) { } // this must fail when it's executed

Widget *pw2 = new (std::nothrow) Widget; // returns 0 if alloc fails
if (pw2 == 0) { } // this might succeed
```

Using `nothrow` guarantees that the `operator new` won't throw, but not the ctor. Just avoid it.

## 50 When necessary, replace `new` and `delete`

### why the black magic

- **To detect usage errors.**
    - Keep track of the numbers of `new`s and `delete`s to prevent memory leak.
    - Prevent data overruns (writing beyond the end of allocated block) and underruns.
        - Custom `new`s can pad a few bytes as signatures, before and after the block.
        - Custom `delete`s can check the signatures and report if they're corrupted.
- **To improve efficiency.**
    - General `new` and `delete` takes a middle-of-the-road strategy.
    - Customizing can help opinionate.
- **To collect usage statistics.**
    - Distribution of block sizes and lifetimes; FIFO, LIFO or random; changes over time; max amount of allocated memory...

### over/underrun detection using signatures

First draft, which is **flawed**.

```cpp
static const int signature = 0xDEADBEEF;
typedef unsigned char Byte;

void* operator new(std::size_t size) throw(std::bad_alloc) {
	using namespace std;

	size_t realSize = size + 2 * sizeof(int);
	void *pMem = malloc(realSize);
	if (!pMem) throw bad_alloc();

	// write sig to beginning and end
	*(static_cast<int*>(pMem)) = signature;
	*(reinterpret_cast<int*>(
			static_cast<Byte*>(pMem) + realSize - sizeof(int))) = signature;

	// return a pointer to just after the first sig
	return static_cast<Byte*>(pMem) + sizeof(int);
}
```

### alignment

- C++ requires all `operator new`s return pointers that are suitably aligned for *any* data type.
- `malloc` guarantees this, so returning a pointer returned by `malloc` is safe.
- But we returned a pointer from `malloc` offset by the size of an `int`.

### alternatives

- Complier switches for debugging and logging functionality.
- Open source or commercial libraries for memory managers.

## 51 Adhere to convention when writing `new` and `delete`

### ROT

- `operator new` should
    - contain infinite loop trying to allocate memory and call new-handler if fail
    - handle requests for zero bytes
    - for member function versions, check for size
- `operator delete` should
    - do nothing if passed `nullptr`
    - for member function versions, check for size

### conventions for`new`

Pseudocode:

```cpp
void* operator new(std::size_t size) throw(std::bad_alloc) {
	using namespace std;
	if (size == 0) {
		size = 1;
	}
	while (true) {
		/* Try to allocate `size` bytes here */

		if (/* allocation successful */) {
			return /* pointer to allocated memory */;
		}
		// alloc failed; find out current new-handler and call
		new_handler globalHandler = set_new_handler(nullptr);
		set_new_handler(globalHandler);
		if (globalHandler) (*globalHandler)();
		else throw std::bad_alloc();
	}
}
```

- there's only one way to get current new-handler
- `operator new` has an infinite loop, so writing valid new-handler is crucial
- `operator new` is inherited by derived classes, but it might be specifically designed for the base class and will break the derived class.
    - Add a comparison for size of the base class.

        `new` with zero bytes will get handled by `::operator new`.

        ```cpp
        void* Base::operator new(std::size_t size) throw(std::bad_alloc) {
        	if (size != sizeof(Base)) return ::operator new(size);
        }
        ```

- For `operator new[]`, do not do anything to the nonexist objects. You don't even know how many objects are there.
    - the operator might be called on derived classes
    - there might be extra space in `size` that's used for tracking number of elements

### conventions for `delete`

C++ guarantees it's always safe to delete the null pointer. Typical pseudocode:

```cpp
void operator delete(void *rawMemory) throw() {
	if (rawMemory == nullptr) return;

	/* deallocate memory pointed to by rawMemory */
}
```

For the member function version, also remember to check correct size:

```cpp
void Base::operator delete(void *rawMemory, std::size_t size) throw() {
	if (rawMemory == nullptr) return;
	if (size != sizeof(Base)) {
		::operator delete(rawMemory);
		return;
	}

	/* deallocate memory pointed to by rawMemory */
}
```

Note: if the base class doesn't have a virtual destructor, the `size` argument might be wrong.

## 52 Write placement `delete` if you write placement `new`

### ROT

- write placement `new`/`delete` in pairs
- be aware of name hiding and provide standard forms

### terminology

If an `operator new` takes extra parameters it's called *placement `new`*; If an `operator delete` takes extra parameters it's called *placement* `delete`.

Placement `new` also sometimes refer to the specific `operator new` introduced in `<new>`: `void* operator new(std::size_t, void* pMem) throw();`.

### compilers taking care of exceptions thrown when `new`ing

For `Widget *pw = new Widget;`, two functions are called: `operator new` and `Widget`'s default ctor. If `operator new` succeeds, memory is allocated; then if the default ctor throws, `operator delete` will be called to clean up the memory and prevent memory leak.

It works out of the box for standard `new` and `delete`, but if you have placement `new`s, you also need to have placement `delete`s that have matching signatures.

### providing full support for `new` and `delete`

- `delete ptr;` never calls placement `delete`s. You need to provide both forms.
- Normal name hiding rules apply to `operator new`s and `operator delete`s as well.
- By default, C++ offers these `operator new`s at global scope:

    ```cpp
    void* operator new(std::size_t) throw(std::bad_alloc);
    void* operator new(std::size_t, void*) throw();
    void* operator new(std::size_t, const std::nothrow_t&) throw();
    ```

Unless you intentionally want to hide some of these standard `new`s, you should provide all of them by forwarding them to the standard `new`/`delete`. 

- Let's put them in a base class so it can be reused.

    ```cpp
    class StandardNewDeleteForms {
    	// normal new/delete
    	static void* operator new(std::size_t size) throw(std::bad_alloc)
    	{ return ::operator new(size); }
    	static void operator delete(void* pMem) throw()
    	{ ::operator delete(pMem); }

    	// placement new/delete
    	static void* operator new(std::size_t size, void* ptr) throw()
    	{ return ::operator new(size, ptr); }
    	static void* operator delete(void* pMem, void* ptr) throw()
    	{ return ::operator delete(pMem, ptr); }

    	// nothrow new/delete
    	static void* operator new(std::size_t size,
    														const std::nothrow_t& nt) throw()
    	{ return ::operator(size, nt); }
    	static void* operator(void *pMem, const std::nothrow_t&) throw()
    	{ return ::operator delete(pMem); }
    };
    ```

- Clients can inherit the class and use `using`, just like in [Item 33](https://www.notion.so/nerrons/Effective-C-73e0037beb024637b5e5e08848f8d4fd#615a8ccb951843248fa2c29d9e333d6e).

    ```cpp
    class Widget : public StandardNewDeleteForms {
    public:
    	using StandardNewDeleteForms::operator new;
    	using StandardNewDeleteForms::operator delete;

    	static void* operator new(std::size_t size,
    														std::ostream& logStream) throw(std::bad_alloc);
    	static void operator delete(void* pMem, std::ostream& logStream) throw();
    };
    ```

# Miscellany

## 53 Pay attention to compiler warnings

- Compiler is probably smarter than you.
- Aim for the max level.

## 54 Familiarize urself with standard libraries

### C++98

- STL: containers, iterators, algorithms, container and function object adapters
- iostreams: user-defined buffering, i18n IO, predefined objects `cin`, `cout`, `cerr`, `clog`
- i18n: multiple active locales, `wchar_t`, `wstring`
- numeric processing: `complex`, `valarray`
- exception hiearchy: base class `exception`, derived `logic_error`, `runtime_error`
- C89 standard library.

### C++0x

- smart pointers
- `function`
- `bind`
- Other discrete functionalities:
    - Hash tables, regex, tuples, `array`, `mem_fn`, `reference_wrapper`, rand, math, C99
- Template programming:
    - type traits, `result_of`.

## 55 Use `Boost`.
