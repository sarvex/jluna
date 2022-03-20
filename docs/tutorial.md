# jluna: A gentle Introduction

This document is meant for those who are unfamiliar with C++ at a high-level, or those just want to use `jluna` for a singular purpose, not needing to make use of all of its more complex functionalities.

This tutorial will teach users the basics but will gloss over much of the implementation details and options, for example, when are multiple ways to do the same thing in `jluna`, this tutorial will only mention the recommended, most convenient way. This often comes at the cost of either performance or control, so if new users find themself tailoring their application more towards either of these goals, it may be necessary to catch up on all `jluna` offers in the manual.

Users who are already familiar with C++ and want a full overview of all of `jluna`s feature from the very beginning are encouraged to [read the manual](./manual.md) instead.

---

# Chapter 0: Creating a Project

To be able to work along with examples given in this tutorial, we'll first need a project that lets us actually run and compile C++ code. To make this as frictionless as possible, `jluna` offers `init.sh`, a bash script that downloads `jluna`, builds it, then creates a project with a working hello world. This section will guide users on how to use it:


### Step 0: Install Dependencies

Install:
+ `clang12+`
+ `julia1.7.0+`
+ `cmake3.12+`

on your system.

### Step 1: `init.sh`

Download the script [here](https://raw.githubusercontent.com/Clemapfel/jluna/master/install/init.sh). Navigate into the download location using a console, then execute:

```bash
# in the same folder as init.sh
./init.sh jluna_tutorial ~/Desktop clang++-12
```

Where `~/Desktop/` will be your directory workspace. Feel free to change the location to anything we want and replace any mention of `~/Desktop/` with your directory henceforth. 

`init.sh` will download `jluna` into the directory `~/Desktop/jluna_tutorial` and build it, if any errors appear, please consult the [installation tutorial](installation.md).

### Step 2: Rebuilding

In your new desktop folder, you will find `~/Desktop/jluna_tutorial/main.cpp`:

```cpp
#include <jluna.hpp>
using namespace jluna;

int main()
{
    State::initialize();
    Base["println"]("Your project is setup and working!");
}
```

As you make changes to this main over the course of the tutorial, you can recompile your application using:

```bash
# in ~/Desktop/jluna_tutorial
cd build
cmake .. -DCMAKE_CXX_COMPILER=clang++-12
make
```

### Step 4: Running your Executable

To execute your executable, you can do:

```bash
# in ~/Desktop/jluna_tutorial/build
./hello_world
```
```
[JULIA][LOG] initialization successfull.
Your project is setup and working!
```

Now that your project is set up, we can go on to actually TODO

## Chapter 1: Calling Julia Code from C++

(code from this tutorial is already available in *your* project in `~/Desktop/jluna_tutorial/examples/TODO.cpp`)

### 1.1 `safe_eval`

When using the Julia REPL interactively, you can run code by inputing it into the terminal and pressing enter. `jluna` offers a similar feature, except instead of the user writing code themselves, the C++-side of your executable will be writing it for you. Instead of pressing enter, you do:

```cpp
State::safe_eval("println(\"julia code here\")");
```
```
julia code here
```

To write multi-line code or code with a lot of character that we would need to escape such as `\"`, we can use the C++ raw string literal `R"()"`, which will treat anything between the brackets as a raw string, automatically escaping any characters that need to be escaped:

```cpp
State::safe_eval(R"(
    println("first line of julia code")
    println("second line of julia code")
)");
```
```
first line of julia code
second line of julia code
```

Using the raw string literal like this is often much more convenient.

Because we are using `safe_eval`, any exception that occurs withing the code that is the string argument will be forwarded to C++ as a `jluna::JuliaException`:

```cpp
State::safe_eval("sqrt(-1)");
```
```
exception in jluna::State::safe_eval for expression:
"sqrt(-1)"

terminate called after throwing an instance of 'jluna::JuliaException'
  what():  [JULIA][EXCEPTION] DomainError with -1.0:
sqrt will only return a complex result if called with a complex argument. Try sqrt(Complex(x)).
Stacktrace:
 [1] throw_complex_domainerror(f::Symbol, x::Float64)
   @ Base.Math ./math.jl:33
 [2] sqrt
   @ ./math.jl:567 [inlined]
 [3] sqrt(x::Int64)
   @ Base.Math ./math.jl:1221
 [4] top-level scope
   @ none:1
 [5] eval
   @ ./boot.jl:373 [inlined]
 [6] eval(m::Module, x::Expr)
   @ Base ./Base.jl:68
 [7] safe_call(::Function, ::Module, ::Expr)
   @ Main.jluna.exception_handler ./none:517
 [8] safe_call
   @ ./none:538 [inlined]
 [9] safe_call(expr::Expr)
   @ Main.jluna.exception_handler ./none:534

signal (6): Aborted
```

### 1.2 Accessing Return Elements

If our julia code returns a result, we can access it by capturing the result of `safe_eval`:

```cpp
Proxy result = State::safe_eval("return 1234");
```

Where `jluna::Proxy` is a stand-in for any julia-side value. Despite the code `return 1234` obviously returning an integer julia-side, `State::safe_eval` always returns a proxy. To transform this proxy into the actual julia-side value, we use *casting*. Casting, in C++, is done using `static_cast<T>` where `T` is the type we want to cast a value to. We can thus transform our result proxy `result` into the actual integer like so:

```cpp
auto integer_result = static_cast<Int64>(result);
```
Where `auto` is a C++ keyword that simplifies variable declaration, because the compiler knows that the result of the `static_cast` will always be an `Int64`, it automatically deduces integer_result to be of type `Int64`.

Now that we transformed it, we can use the result just like any integer:

```cpp
std::cout << integer_result + 1111 << std::endl;
```
```
2345
```
Where `std::cout << value << std::endl` is the C++ equivalent of julias `println(value)`.

### 1.3 Accessing Julia-Side Variables

We learned how to access the return value of any piece of code, however, that's not the only value a script can set. Consider the following:

```cpp
auto res = State::safe_eval(R"(
    variable = 1234;
    return "string"
)");
```

Where `auto` is deduced to `jluna::Proxy` and `R"()"` declared the code in between the brackets to be treated as a raw string.

We know that `res` now holds the return result `"string"`, however this piece of code furthermore created a new variable `variable` julia-side, that holds the value `1234`. To access it, we could do:

```cpp
auto other_res = State::safe_eval("return variable");
```

However, `jluna` offers a much more elegant solution:

```cpp
auto other_res = Main["variable"];
```

Let's look at this statement piece-by-piece. 

Firstly, `auto other_res` declares a C++ variable, named `other_res`, that is, again, a `jluna::Proxy`, just like with `State::safe_eval`. It will point to the result of the expression right of the `=`. 

There, we first have a globally set `jluna` variable called `Main`. This variable is of type `jluna::Module` and holds the julia-side module of the same name. Along with main, `jluna` also automatically sets `Base` and `Core`. 

Lastly we have the `["variable"]`. In C++, this is called an invocation of `operator[]`. It is best to think of `operator[]` as `jluna`s equivalent of the julia-side dot operator `.`:

```julia
# in julia:
Main.variable

# in cpp:
Main["variable"]
```

Therefore, calling `Main["variable"]` access the variable named `variable` in the julia-side module `Main` and returns a `jluna::Proxy` holding the result.

To make the result available as the actual type it is julia-side (recall that we declared `variable = 1234`), we cast it again, just like we did before:

```cpp
auto res = State::safe_eval(R"(
    variable = 1234;
    return "string"
)");
auto other_res = Main["variable"];

std::cout << static_cast<std::string>(res) << std::endl;
std::cout << static_cast<Int64>(other_res) << std::endl;
```
```
1234
string
```

### 1.4 Calling Julia Functions

So far we've called julia code by running it as a string, this method has several downsides, however. Most importantly, it is slow. Anytime we execute a string, julia-side `Meta.parse` is called to parse the string, then `Base.eval` is called to evaluate the resulting expression. To avoid introducing this overhead anytime we want to execute code that exists in the julia state, we should instead do it the same way we would do so in julia: calling it as a function.

Unlike in C++, in julia, any function is just a regular object. We can move it, assign it to a variable, etc. Because of this, its best to think of julia-functions like variables in module scope, for example the *code* function `Base.println` is bound to a variable name `println` in the Module `Base`. Because of this, we can actually access julia-side functions just like we did variables before:

```cpp
auto println = Base["println"];
```
Where we again use `Base.operator["println"]` as the `jluna`/C++ equivalent of what would be `Base.println`, in julia.

To call the now C++-side `println`, we use `()`, just like we would in julia. Just like calling `[]` is called `operator[]` in C++, using `()` is called `operator()`.

To call `println` with C++-side values, we simply call it just like we would any function.

```cpp
println()
```
```

```
Which, of course, prints just an empty line. Note that, here, we are indeed invoking the julia-side function and running julia-side code, because of `jluna`s design using `operator()` on a variable that points to a julia-side function automatically calls the julia-side function directly.

Only printing an empty line isn't very useful, of course. Calling `println` with C++-side arguments works with no further user interaction:

```cpp
auto println = Base["println"];
std::string cpp_side_string = "it just works";
println(cpp_side_string);
```
```
it just works
```

Most C++ standard library types are supported to be able to be directly used like this. A full list is of supported types is available [here](manual.md/#list-of-unboxables).

The true power of using `jluna`s proxy-class (just like before, `println` is a proxy)  to call julia-side functions is that we can mix C++-side and julia-side values when using them as arguments:

```cpp
auto println = Base["println"];
std::string cpp_side_string = "it just works";
auto julia_side_string = State::safe_eval("return \"just like this\"");
println(cpp_side_string, julia_side_string);
```
```
it just works
just like this
```

A word on performance, when we call julia-side functions with julia-side values, performance is optimal. Invoking `operator()` on the C++-side proxy is exactly as fast as calling the function the proxy points to with the julia-side argument directly in julia. If the argument is not julia-side, however, we usually first need to move it there. 

So in the statement:

```cpp
println(cpp_side_string, julia_side_string);
```
`cpp_side_string` first needs to be moved to the julia-state before `println` can be called. This may cause unintended performance overhead. To fix this, simply make sure to only use julia-side values as arguments. What if we have to use C++-side values though, you may ask? Well...

---

## Chapter 2: Accessing C++-side Values from Julia

We should introduce a concept that is fundamental to how `jluna` transfers values between states: *boxing* and *unboxing*. 
+ **boxing** means to take a C++-side value, convert it to a julia-compatible memory format (if necessary) and move it to the julia-state
+ **unboxing** means to take a Julia-side value, convert it to a C++-compatible memory format (if necessary) and move it to the C++-state

To box the string `"string"`, we do:

```cpp
auto* julia_side_string = box<std::string>(string);
```

Where `auto*` is deduced to type `Any*`, which is best thought of as a pointer to any julia-side object (but not necessarily an object of type `Base.Any`).

To unbox the now julia-side string, we do:

```cpp
auto back_cpp_side_string = unbox<std::string>(julia_side_string);
```

Where `auto` is deduced to `std::string`

The template argument inside the `<>` needs be explicitly specified only for `unbox` (though it is good style to do so for both).

### 2.1 Moving C++-side Values to Julia













