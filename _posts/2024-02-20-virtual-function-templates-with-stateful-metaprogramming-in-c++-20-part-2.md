---
layout: post
title: "Virtual function templates with stateful metaprogramming in C++ 20: Part 2"
date: 2024-02-20
---

In [part 1](2024-02-18-virtual-function-templates-with-stateful-metaprogramming-in-c++-20.md) of this series we learned how to implement a virtual function template with a variadic parameter pack. In this post we're going to expand on our code to allow for an arbitrary number of virtual function templates with different return types. And we'll do it all with one single vtable!

## Generalizing the vtable functions

Our current `vtable_func` implementation looks like this:

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

Right now it's limited to calling the `print` function on the target object, but with some straightfoward changes we can make it able to call any member function. To start off we're going to rename the template parameter pack on`vtable_func` from `Args` to a more generic name `Ts`. The reason is because `Ts` is now going to represent the types of the template arguments for whatever function we're calling, and those are not necessarily the same thing as the types of the arguments to the function. For example:

```cpp
template <typename... Ts>
void f(int i, double d, Ts&&... ts);

template <typename... Ts>
void g();
```

Here we have two different functions with template parameter packs `Ts`. The function `f` takes arguments of type `(int, double, Ts&&...)` and the function `g` takes no arguments. We need to be able to handle virtual functions like these where the types of the template arguments don't necessarily correspond to the types of the function arguments.

The second thing we'll do is add a new `run_impl` function to `vtable_func` that will help us generalize `run`. We can even make this new function private as a matter of good practice. `run` will pass its arguments to `run_impl`, but will additionally pass a pointer-to-member-function that is intended to be called. The return type `R` and parameter types `Args` of the member function can be deduced by the compiler.

```cpp
template <typename Derived, typename R, typename... Args>
static void run_impl(R(Derived::* func)(Args...), Printer* printer, void* argsTuplePtr)
{
    const auto bound = std::bind_front(func, static_cast<Derived*>(printer));

    auto& argsTuple = *static_cast<std::tuple<Args&&...>*>(argsTuplePtr);

    std::apply(bound, std::move(argsTuple));
}
```

`run_impl` looks very similar to our original `run` function except for the extra `R(Derived::* func)(Args...)` parameter. This is a parameter named `func` that is a pointer to a member function of the `Derived` class. It takes arguments of type `Args...` and has return type `R`. The compiler is able to deduce `Derived`, `Args` and `R` based on whatever function we give to `run_impl`. For simplicity I've replaced the `bound` lambda with `std::bind_front` which does the same thing in only one line of code. Everything else is the same as before: we cast `argsTuplePtr` to the correct tuple type and then use `std::apply` to call `bound` with the arguments.

The `run` function itself becomes simple and our whole `vtable_func` struct looks like this:

```cpp
template <typename... Ts>
struct vtable_func
{
private:
    template <typename Derived, typename R, typename... Args>
    static void run_impl(R(Derived::* func)(Args...), Printer* printer, void* argsTuplePtr)
    {
        const auto bound = std::bind_front(func, static_cast<Derived*>(printer));

        auto& argsTuple = *static_cast<std::tuple<Args&&...>*>(argsTuplePtr);

        std::apply(bound, std::move(argsTuple));
    }

public:
    template <typename Derived>
    static void run(Printer* printer, void* argsTuplePtr)
    {
        run_impl(&Derived::template print<Ts...>, printer, argsTuplePtr);
    }
};
```

`run` passes a pointer to whatever member function we want to call (currently just `Derived::print<Ts...>`) along with the pointer to the `Printer` and the pointer to the arguments tuple. `run_impl` is then responsible for invoking that function on the printer.

## Adding a new virtual function template

Now we're able to run any arbitrary member function we want just by passing it to `run_impl`. We already have a `print` function that prints the arguments to `std::cout`. Let's add a new `print_to_stream` function that will let us print to any `std::ostream`. First we'll add the implementation to `PrinterImpl`.

```cpp
template <typename... Args>
void print_to_stream(std::ostream& stream, Args&&... args)
{
    ((stream << args << '\n'), ...);
}
```

Inside `vtable_func::run` we can invoke this new function just like we invoked `print`.

```cpp
run_impl(&Derived::template print_to_stream<Ts...>, printer, argsTuplePtr);
```

But wait a minute! How is the `run` function supposed to know what function to call? It currently doesn't have enough information to know whether it should run `print` or `print_to_stream`. How do we fix this? With another template parameter of course!

Let's define an enum:

```cpp
enum class Function
{
    Print,
    PrintToStream,
};
```

`vtable_func` will take a `Function` as a non-type template parameter in addition to the `Ts` parameter pack.

```cpp
template <Function F, typename... Ts>
struct vtable_func
```

`run` uses it to determine which member function pointer it needs to pass to `run_impl`.

```cpp
template <typename Derived>
static void run(Printer* printer, void* argsTuplePtr)
{
    if constexpr (F == Function::Print)
    {
        run_impl(&Derived::template print<Ts...>, printer, argsTuplePtr);
    }
    else if constexpr (F == Function::PrintToStream)
    {
        run_impl(&Derived::template print_to_stream<Ts...>, printer, argsTuplePtr);
    }
}
```

The final step is to use the correct `Function` case when pushing the `vtable_func` into the `stateful_type_list`. The original `Printer::print` function will now look like this:

```cpp
template <typename... Args,
          size_t Index = stateful_type_list::try_push<vtable_func<Function::Print, Args...>>()>
void print(Args&&... args)
```

And `Printer::print_to_stream` is implemented like so:

```cpp
template <typename... Args,
          size_t Index = stateful_type_list::try_push<vtable_func<Function::PrintToStream, Args...>>()>
void print_to_stream(std::ostream& stream, Args&&... args)
{
    auto argsTuple = std::forward_as_tuple(stream, std::forward<Args>(args)...);

    m_vtable[Index](this, &argsTuple);
}
```

## Testing it out

Let's try out our new virtual function template! In `main` we'll call `print_to_stream` instead of print and see what happens.

```cpp
auto p = make_printer();

double d = 2.5;
const std::string s = "Hello, world!";

p->print_to_stream(std::cerr, 5, d, s);
```

The output prints to `cerr` like we expect.

```
5
2.5
Hello, world!
```

Any number of virtual function templates can be supported by adding a new `Function` case and using it accodingly.

# Supporting return values

We can have as many virtual function templates as we want, so let's figure out how to support returning values from these functions. We'll add a third function to our `PrinterImpl` class called `print_to_string`. We also need to add a corresponding `PrintToString` case to the `Function` enum.

```cpp
template <typename... Args>
std::string print_to_string(Args&&... args)
{
    std::stringstream stream;

    ((stream << args << '\n'), ...);

    return stream.str();
}
```

We want to return a `std::string` from `Printer::print_to_string` but this is complicated by the fact that we use a single vtable for all of our functions and, in general, each function might have a different return type. One solution could be to have `vtable_func::run` return a `std::variant` of all the possible return types (using `std::monostate` to represent `void`) and then use `std::get` to grab the correct type in our `Printer` implementation. This would work just fine but it is potentially inefficient. Let's say that we have 10 virtual function templates and nine of them return small types like `int` or `char` but one of them returns a `std::array<int, 1000>`. Because `std::variant` must have a size that is _at least_ as big as its largest possible type, this will force all of our functions to have a large return type and lead to unnecessay copying of data.

The more efficient solution is to have the caller of the vtable function provide the storage for the return type and then pass a pointer to that storage into the vtable function. We'll add a `void* ret` parameter to `vtable_func::run` and pass that through to `run_impl`.

```cpp
template <typename Derived>
static void run(Printer* printer, void* argsTuplePtr, void* ret)
{
    if constexpr (F == Function::Print)
    {
        run_impl(&Derived::template print<Ts...>, printer, argsTuplePtr, ret);
    }
    else if constexpr (F == Function::PrintToStream)
    {
        run_impl(&Derived::template print_to_stream<Ts...>, printer, argsTuplePtr, ret);
    }
    else if constexpr (F == Function::PrintToString)
    {
        run_impl(&Derived::template print_to_string<Ts...>, printer, argsTuplePtr, ret);
    }
}
```

Inside `run_impl` we need to move the result of calling `func` into the storge provided by `ret`. We'll use a `std::optional` for the storage, so `run_impl` will cast `ret` from a `void*` to a `std::optional<R>*` and assign to it. We also need to have special logic for the case where the return type is `void` because we can't assign `void` to anything or create a `std::optional<void>`.

```cpp
template <typename Derived, typename R, typename... Args>
static void run_impl(R(Derived::* func)(Args...), Printer* printer, void* argsTuplePtr, void* ret)
{
    const auto bound = std::bind_front(func, static_cast<Derived*>(printer));

    auto& argsTuple = *static_cast<std::tuple<Args&&...>*>(argsTuplePtr);

    if constexpr (std::is_same_v<R, void>)
    {
        std::apply(bound, std::move(argsTuple));
    }
    else
    {
        *static_cast<std::optional<R>*>(ret) = std::apply(bound, std::move(argsTuple));
    }
}
```

`Printer::print` and `Printer::print_to_stream` will be updated to pass a `nullptr` as the `ret` argument to the vtable function. This is safe because both of these functions return `void` so `vtable_func::run_impl` won't attempt to use the pointer. `Printer::print_to_string` is implemented like so:

```cpp
template <typename... Args,
          size_t Index = stateful_type_list::try_push<vtable_func<Function::PrintToString, Args...>>()>
std::string print_to_string(Args&&... args)
{
    auto argsTuple = std::forward_as_tuple(std::forward<Args>(args)...);

    std::optional<std::string> ret;

    m_vtable[Index](this, &argsTuple, &ret);

    return std::move(*ret);
}
```

Let's verify that it works.

```cpp
auto p = make_printer();

double d = 2.5;
const std::string s = "Hello, world!";

const auto str = p->print_to_string(5, d, s);

std::cout << str;
```

The output is what we expect.

```
5
2.5
Hello, world!
```

Play around with the code here: [https://godbolt.org/z/Gsr1EKavz](https://godbolt.org/z/Gsr1EKavz)

View it on my github: [https://github.com/christiandaley/examples/blob/main/cpp/virtual_function_templates-part-2/main.cpp](https://github.com/christiandaley/examples/blob/main/cpp/virtual_function_templates-part-2/main.cpp)

## Final remarks

We've seen that supporting an arbitrary number of virtual function templates and different return types is actually quite straightforward. The amount of new code we added was pretty small and the only new type introduced was the `Function` enum.

It should be noted that our existing implementation does not support returning reference types because `std::optional` cannot contain a reference. This limitation is easily overcome by having an additional `if constexpr` case in `run_impl` to handle this case and using a raw pointer instead of a `std::optional` to temporarily store the return value. The implementation of this is left as an exercise for the reader.

## What's next?

At the end of part one I mentioned that these virtual function templates are unsafe to use across translation units because of ODR violations that result from stateful metaprogramming. I also teased that it may be possible to get around this limitation, and in part 3 that's exactly what we'll do.
