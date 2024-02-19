---
title: "Virtual function templates with stateful metaprogramming in C++ 20"
date: 2024-02-18
---

Virtual function templates are not possible in C++. Go ahead, try it yourself. No compiler will accept the following code:

```cpp
struct Base
{
    template <typename T>
    virtual void do_something(T arg) = 0;

    virtual ~Base() = default;
};

struct Derived : Base
{
    template <typename T>
    void do_something(T arg) override
    {
        // do something
    }
};
```

The reason for this is simple: `do_something` _is not a function_, it is a function template. Because the compiler operates on translation units (source files) independently of each other, there's no possible way for it to know which specializations of `print` need to be instantiated and included in the vtable of `Base` and `Derived`. Assume that these structs are declared in some header file `header.h`, then consider the following code:

```cpp
// foo.h
#include "header.h"
void foo(Base& base);
```

```cpp
// foo.cpp
#include "foo.h"

void foo(Base& base)
{
    base.do_something(5);
}
```

```cpp
// main.cpp
#include "foo.h"

int main()
{
    Derived derived{};
    foo(derived);
}
```

Here we attempt to create an instance of `Derived` in our `main.cpp` file and pass it to a function that's implemented in `foo.cpp`. In `foo.cpp` we invoke `do_something<int>`. The issue is that when `derived` is constructed in `main.cpp` the compiler must know exactly which specializations of `do_something` are needed so that they can be included in `derived`'s vtable. But in order to get that information the compiler would need to look at `foo.cpp` while in the middle of compiling `main.cpp` (and, in general, would also need to examine every single other source file in the program) to see that `do_something<int>` is needed. C++ compilers cannot do this, and thus virtual function templates are not allowed.

It would be more accurate to say that virtual function templates are not part of the C++ standard and thus compilers don't need to do that sort of thing that would definitely be horrendous for compile times. My point is that there is no way to create virtual function templates in C++.

...

Or is there?

## The goal

Forget trying to make virtual function templates work across translation units. In this post we're going to focus on achieving virtual function templates within the scope of a single source file. By the end of this post I'll show you that implementing the following code is completely possible by using some C++ black magic:

```cpp
#include <iostream>

class Printer
{
public:
    template <typename... Args>
    void print(Args&&... args)
    {
        // ???
    }

    virtual ~Printer() = default;
};

class PrinterImpl : Printer
{
public:
    template <typename... Args>
    void print(Args&&... args)
    {
        ((std::cout << args << '\n'), ...);
    }
};

std::unique_ptr<Printer> make_printer()
{
    return std::make_unique<PrinterImpl>();
}

int main()
{
    auto p = make_printer();

    double d = 2.5;
    const std::string s = "Hello, world!";

    // calls PrinterImpl::print<int, double&, const std::string&> !!
    p->print(5, d, s);
}
```

## Where to start?

If the compiler is unwilling to build a vtable for us, then we'll just have to do it ourselves.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ca80m4w6fd23rnvd3xvc.jpg)

But what should our vtable entries look like? I propose the following function signature:

`void(*)(Printer*, void*)`

In short, a vtable entry will be a function that takes a pointer to a `Printer` (the object that `print` is being called on) and a pointer to the arguments in the form of a `void*`. The vtable itself needs to be a collection of such functions, so our `Printer` class will have a member:

`const std::span<void(*)(Printer*, void*)> m_vtable;`

With that in mind, our implementation of `Printer::print` will look something like this:

```cpp
template <typename... Args>
void print(Args&&... args)
{
    auto argsTuple = std::forward_as_tuple(std::forward<Args>(args)...);

    size_t vtableIndex = ???

    m_vtable[vtableIndex](this, &argsTuple);
}
```

All of the arguments are combined into a `std::tuple` so that they can be passed into the vtable function via a single void pointer. `std::forward_as_tuple` is used to preserve the reference type of each argument. Of course, this still leaves three massive questions that need to be answered:

1. Where are these vtable functions defined?
2. How is the index into the vtable for each specialization of `Printer::print` determined?
3. How is the `m_vtable` member created?

## Defining the vtable functions

To start off we'll implement the vtable functions we need.

```cpp
template <typename... Args>
struct vtable_func
{
    template <typename Derived>
    static void run(Printer* printer, void* argsTuplePtr)
    {
        const auto bound = [&](Args&&... args)
        {
            static_cast<Derived*>(printer)->print(std::forward<Args>(args)...);
        };

        auto& argsTuple = *static_cast<std::tuple<Args&&...>*>(argsTuplePtr);

        std::apply(bound, std::move(argsTuple));
    }
};
```

Let's go through this line by line:

1. We start off by declaring a new struct template called `vtable_func` that is templated on a parameter pack `Args`. As you can probably guess, `Args` corresponds to the types of arguments passed to `Printer::print`. It's not shown here but the definition of `vtable_func` exists within the scope of the `Printer` class.
2. `vtable_func` has a static function template called `run`. The `Derived` type parameter here represents the concrete, underlying type of the `Printer` object. In our code this will end up being `PrinterImpl`.
3. Inside `run` we create a lambda called `bound`. This lambda takes the expected arguments and forwards them to the implementation of `print` for the `Derived` type. We could have also used `std::bind_front` here.
4. We cast `argsTuplePtr` to a reference to its underlying type `std::tuple<Args&&...>`
5. We use `std::apply` to apply all of the arguments to our `bound` lambda, thus calling the desired `print` function with all of the expected arguments.

When `Printer::print` is called with some set of arguments of type `Args&&...` on some object of type `Derived` that inherits from `Printer`, we need to invoke the corresponding `vtable_func<Args...>::run<Derived>` function.

You may be wondering why I chose to have the `Args` parameter pack be defined on a struct template while the `Derived` template parameter is defined on the inner `run` function. Why not just get rid of the struct and have a function that looks like this:

```cpp
template <typename Derived, typename... Args>
void run(Printer* printer, void* argsTuple)
{...
```

The need for separating the `Args` and `Derived` template parameters will become clear soon.

## Getting the vtable index

So, we've written our vtable functions. On to the next challenge: how do we determine what index in the vtable corresponds to any particular specialization of `Printer::print`? After all, `Printer::print<int, double>` will need to have a different vtable entry than `Printer::print<double, int>` and we need to make sure that every specialization will end up calling the correct function.

Enter stateful metaprogramming.

### Stateful metaprogramming

I could write multiple posts entirely about how stateful metaprogramming works, but there are already quite a few such articles scattered across the internet. If you're not already familiar with stateful metaprogramming I suggest you start by reading these excellent articles:

1. [https://mc-deltat.github.io/articles/stateful-metaprogramming-cpp20](https://mc-deltat.github.io/articles/stateful-metaprogramming-cpp20)
2. [https://b.atch.se/posts/constexpr-counter/](https://b.atch.se/posts/constexpr-counter/)
3. [https://b.atch.se/posts/constexpr-meta-container/](https://b.atch.se/posts/constexpr-meta-container/)

With that said, I'd like to introduce a very simple `type_list` implementation:

```cpp
template <typename... Ts>
struct type_list
{
    template <typename... Us>
    constexpr auto operator+(type_list<Us...>) const noexcept
    {
        return type_list<Ts..., Us...>{};
    }

    template <typename... Us>
    constexpr bool operator==(type_list<Us...>) const noexcept
    {
        return std::is_same_v<type_list<Ts...>, type_list<Us...>>;
    }
};
```

It's just a simple struct that's templated on a parameter pack `Ts`. It serves one purpose and that is to "store" a list of types. `type_list` has a `+` operator that allows for concatenation of lists, and I also went ahead and defined a `==` operator that is able to conveniently tell us if two lists are equal. We can use these operators like so:

```cpp
static_assert(type_list<int>{} + type_list<char>{} == type_list<int, char>{});

static_assert(type_list<>{} + type_list<double>{} == type_list<double>{});

static_assert(type_list<int, double>{} != type_list<double, int>{});
```

Now we'll use our `type_list` struct to implement a stateful type list:

```cpp
struct stateful_type_list
{
private:
    template <size_t N>
    struct getter
    {
        friend consteval auto flag(getter);
    };

    template <typename T, size_t N>
    struct setter
    {
        friend consteval auto flag(getter<N>)
        {
            return type_list<T>{};
        }

        static constexpr size_t value = N;
    };

    template <typename T, size_t N = 0>
    consteval static size_t try_push()
    {
        if constexpr (requires { flag(getter<N>{}); })
        {
            return try_push<T, N + 1>();
        }
        else
        {
            return setter<T, N>::value;
        }
    }

public:
    template <typename Unique, size_t N = 0>
    consteval static auto get()
    {
        if constexpr (requires { flag(getter<N>{}); })
        {
            return flag(getter<N>{}) + get<Unique, N + 1>();
        }
        else
        {
            return type_list{};
        }
    }
};
```

And we can use it like so:

```cpp
static_assert(stateful_type_list::get<decltype([]{})>() == type_list{});

static_assert(stateful_type_list::try_push<int>() == 0);

static_assert(stateful_type_list::try_push<double>() == 1);

static_assert(stateful_type_list::get<decltype([]{})>() == type_list<int, double>{});
```

I'm not going to fully explain how this code allows for state to be stored and retrieved at compile time as I'm assuming that you’ve already made yourself familiar with the posts I linked above. I will however explain several peculiarities in my implementation:

1. The `try_push` function returns the index in the list at which the new type was added. Types are always added at the end of the list.
2. We are unable to push the same type more than once. You can play around with it yourself to verify. This is because compilers memoize the results of `consteval` functions. We could get around this by also requiring an additional "Unique" template parameter each time we use the `try_push` function, but as it turns out we don't need to be able to push the same type more than once for our vtable implementation.
3. The `get` function requires a unique type to be supplied as a template argument each time it's called in order to avoid the aforementioned memoization. It forces the compiler to re-evaluate the `get` function each time we use it. That is what the `decltype([]{})` in the above code is doing, as each lambda declaration is guaranteed to be a unique type.

### Lambdas as NTTPs

I'd like to mention something important with regards to point number 3 above. It may be tempting to do something like this in order to alleviate the need to manually supply a new type each time you call `get`:

```cpp
template <auto Unique = []{}, size_t N = 0>
static consteval auto get()
```

This actually works in practice (msvc, clang, and gcc all compile this as of 2/18/2024), but after some research I've found that the status of lambdas as non-type template parameters in C++20 and C++23 is a bit murky. Additionally it's not at all clear that the C++ standard would require this default template parameter to be re-evaluated at each call site. For example, a default template parameter of `auto Line = std::source_location::current().line()` will always be the line number of the template declaration, whereas an identical _default argument_ will always be the line number of the call site. This suggests that a default template parameter is not required to behave as if it was explicitly provided at the call site. I want to avoid any sort of implementation defined behavior for the purpose of this post so I won't be using this trick.

### Finishing the `Printer::print` implementation

With the help of our `stateful_type_list` we can finally finish implementing `Printer::print`.

```cpp
template <typename... Args,
          size_t Index = stateful_type_list::try_push<vtable_func<Args...>>()>
void print(Args&&... args)
{
    auto argsTuple = std::forward_as_tuple(std::forward<Args>(args)...);

    m_vtable[Index](this, &argsTuple);
}
```

`vtable_func<Args...>` is pushed into our `stateful_type_list` and the returned `Index` is used to lookup the corresponding function in the vtable. It's critically important that `Index` appears as a default template argument rather than being computed as part of the function body because this forces the compiler to compute the value of `Index` (and hence store `vtable_func<Args...>` in our stateful list) _at the call site_ of Printer::print. The actual code generation of any particular specialization of `Printer::print` could happen at a later time in the compilation phase but we want our `vtable_func` pushed into our `stateful_type_list` ASAP.

We've accomplished two out of our three goals. The last thing we need to do is initialize the `m_vtable` member of our `Printer` class.

## Creating the vtable

In order to initialize our vtable we'll start by writing a function that is capable of creating it. It will be a static function template that exists within the `Printer` class.

```cpp
template <typename Derived, typename... Funcs>
static auto create_vtable(type_list<Funcs...>)
{
    static constinit std::array vtable { Funcs::template run<Derived>... };

    return std::span{ vtable };
}
```

Let's break this down:

1. The template parameter `Derived`, as in other cases, corresponds to the underlying type of the `Printer`.
2. The parameter pack `Funcs...` will be our `vtable_func`s that we pushed into our `stateful_type_list` in the `Printer::print` implementation.
3. The parameter of type `type_list<Funcs...>` is not named and not used. It exists so that the compiler can deduce the `Funcs...` parameter pack.
4. We create a static `std::array` of vtable entries, using CTAD so that we don't need to write `std::array<void(*)(Printer*, void*), void*>, sizeof...(Funcs)>`.
5. The `vtable` array is initialized with the function pointers `run<Derived>` for each `vtable_func` in `Funcs`. The `template` keyword is needed here to make the compiler happy.
6. We return a `std::span` that points to the array. This is safe because the array is static and will exist for the duration of the program.

For any index `i`, the function `m_vtable[i]` will be the correct function to handle the arguments passed to it by the `Printer::print` specialization that pushed a `vtable_func` at index `i` in the `stateful_type_list`. We can now utilize `create_vtable` within a protected constructor of `Printer`.

```cpp
    template <typename Derived>
    Printer(Derived*) :
        m_vtable{ create_vtable<Derived>(stateful_type_list::get<Derived>()) }
    {}
```

1. The constructor has a template parameter `Derived` because the `Printer` needs to know its underlying type when it's creating the vtable. An unnamed dummy parameter of type `Derived*` is required so that the compiler can deduce the `Derived` type.
2. The `type_list<Funcs...>` argument passed to `create_vtable` is obtained by calling `stateful_type_list::get` to get the list of vtable functions. `Derived` is passed as the dummy "Unique" template argument to `get` to ensure that the compiler always re-evaluates `get` for each different child class of `Printer` that is used.

And that's it! The `Printer` class now has a constructor that initializes the vtable. The very last thing we need to do is call this constructor from our `PrinterImpl` default constructor.

```cpp
PrinterImpl() :
    Printer{ this }
{}
```

Whew! We did it!

### Does it work?

After all that effort we'd like to know if our code works! Let's run the following code that I showed at the beginning of this post:

```cpp
std::unique_ptr<Printer> make_printer()
{
    return std::make_unique<PrinterImpl>();
}

int main()
{
    auto p = make_printer();

    double d = 2.5;
    const std::string s = "Hello, world!";

    p->print(5, d, s);

    return 0;
}
```

The output of the program when compiled with gcc 13.2, clang 17.01, and msvc 19.39 (all in c++ 20 mode) is:

```
5
2.5
Hello, world!
```

I'll leave it as an exercise for the reader to verify that all the arguments have their reference types preserved when being passed through the vtable function and into `PrinterImpl::print`. You can play around with the code here: [https://godbolt.org/z/jPbe3hnh4](https://godbolt.org/z/jPbe3hnh4)

You can also find the code on my github: [https://github.com/christiandaley/examples/blob/main/cpp/virtual_function_templates/main.cpp](https://github.com/christiandaley/examples/blob/main/cpp/virtual_function_templates/main.cpp)

To the best of my knowledge, the code shown here is well formed and does not contain any undefined or implementation defined behavior. That being said, I am not an expert when it comes to the C++ standard so it's entirely possible that I'm wrong about that, and if I am I would love if someone was able to point that out. **Regardless, this code should be seen as a curiosity and a learning experience and I absolutely do not recommend using this in any production code.**

## Moving forward

We've shown that, with some boilerplate, virtual function templates are indeed possible in C++ as long as we confine ourselves to a single source file. It's easy to see that our current implementation is pretty limited: we support only a single function template that doesn't return anything. What if we wanted to allow for an arbitrary number of virtual function templates to be declared and implemented? And what if each of those returns a potentially different type? It'd be great if we were able to do something like this:

```cpp
struct Base
{
    template <typename... Args>
    int virtual_func_one(Args&&...)
    {
        // ...
    }

    template <typename... Args>
    std::string virtual_func_two(Args&&...)
    {
        // ...
    }

    template <typename... Args>
    std::vector<double> virtual_func_three(Args&&...)
    {
        // ...
    }
};
```

You might think that we'd need a separate vtable for each of them, but as it turns out we don't! In a future post we'll explore how we can enable this functionality by modifying our existing code.

## Some notes about ODR

C++ has something called the [One Definition Rule](https://en.cppreference.com/w/cpp/language/definition), which basically says that non-inline classes, functions, and variables can have exactly _one definition_ throughout the entire program. Variables/functions marked `inline` and class/function templates can be defined more than once as long as each definition occurs in a different translation unit and **all definitions are identical**. Any violation of this rule results in the program being ill formed, and there is no guarantee that the compiler will issue any sort of error. Consider the following function declared in some file `f.h`:

```cpp
template <typename T = ::Tag>
int f()
{
    return 1;
}
```

This function template `f` has a default template parameter of type `::Tag`, but this type isn't defined in `f.h`. Now consider the following two source files:

```cpp
// A.cpp
namespace
{
    struct Tag {};
}

#include "f.h"
```

```cpp
// B.cpp
namespace
{
    struct Tag {};
}

#include "f.h"
```

Both `A.cpp` and `B.cpp` declare a struct `Tag` within an anonymous namespace prior to including `f.h`. This means that the two versions of `f` that end up being defined in our program will "see" different types as their default template parameter and therefore have different definitions. This violates the ODR rule and such a program is ill formed regardless of whether any existing compiler implementation is willing to compile it. What does this mean for us?

I mentioned earlier that our code only works within a single source file. It should be fairly straightforward to understand the most obvious issue with putting our `Printer` class in a header file and attempting to use any given `Printer` instance across source files. A `PrinterImpl` object constructed in some file `A.cpp` will contain a vtable that is capable of handling only the specific specializations of `Printer::print` that are needed in that one file. Casting it to a `Printer*` and passing it to a function defined in some other source file `B.cpp` is a recipe for disaster. Unless both `A.cpp` and `B.cpp` use the exact same specializations of `Printer::print` _in the exact same order_, we're going to run into problems when `B.cpp` expects a different vtable than the one constructed in `A.cpp`.

We might think to remedy this issue by replacing our array based implementation of the vtable with a `std::unordered_map<std::type_index, void(*)(Printer*, void*)>`. In `Printer::print` we would no longer need to worry about the index returned from `stateful_type_list::try_push` and instead we could use `typeid(vtable_func<Args...>)` to lookup the correct function in the map. Likewise in `create_vtable` we would initialize this map with {% raw %} `{{ typeid(Funcs), Funcs::template run<Derived> }...}` {% endraw %}. This would make it so that the order in which `Printer::print` specializations are used no longer matters, and would also allow us to detect when we've been given a `Printer` that isn't capable of handing a particular set of arguments. In `Printer::print` we could verify that the corresponding entry exists in the map before attempting to call the function and if it doesn't we can do something like throw an exception that could be handled by the caller. This should work great, right?

Wrong.

I don’t know exactly what any particular compiler will do when given such a program, but I can show that such a program violates the ODR and is ill formed.

Going back to the implementation details of `stateful_type_list` let's consider what happens when `print(5)` is called on some `Printer` object. Let's also assume that this is the first and only use of `Printer::print` within this source file. When `vtable_func<int>` is added to the stateful list it results in a declaration of `flag(getter<0>)` being instantiated by the compiler with a deduced return type of `type_list<vtable_func<int>>`. Later, this function is called by `get` in `stateful_type_list`. Friend functions exist at the _namespace scope_ even if they are defined within a struct/class. That means that the compiler sees a free function named `flag` that accepts a parameter of type `getter<0>` and returns a `type_list<vtable_func<int>>`.

Now what happens if we do the same thing in another source file, but instead we call `print(5.0)`. Following the same logic as before, the result will be the declaration of a free function named `flag` that accepts a parameter of type `getter<0>` and returns a `type_list<vtable_func<double>>`. C++ functions cannot be overloaded solely on their return type, so this is in fact a conflicting definition of `flag(getter<0>)`. ODR is violated and the program is ill formed.

This isn't the only ODR violation that could happen. For example, let's say that both our `Printer` and `PrinterImpl` classes are placed in a header file `printer.h`. Remember that the `Printer` constructor will call `create_vtable` which is a function template. The specialization of `create_vtable` that is invoked will depend on the current state of `stateful_type_list`. If two different source files both construct a `PrinterImpl` and call different specializations of `Printer::print` then the constructor `Printer::Printer<PrinterImpl>` will call different specializations of `create_vtable` depending on which source file it's being called in. The result would be two conflicting definitions of the constructor. Again, ODR is violated.

What can we conclude from all this? First of all it's plain to see that stateful metaprogramming is a (potentially) very dangerous practice that can easily lead to ill formed programs and undefined behavior. We can also see that our `Printer` and `PrinterImpl` classes are not safe to use outside of a single source file, and I don't know how useful it is to have virtual function templates that are confined to a single source file. It'd be awesome if we could use this technique across many source files within a single program but the need for meta state will necessarily lead to ODR violations because different state is needed in different translation units. Our code is an interesting novelty but is forever limited by the ODR imposed on us by the C++ standard.

Sadly, there is no way to make our virtual function templates safe to use across translation units.

...

Or is there?
