# What is constant evaluation? #

Constant evaluation is basically just performing calculations at compile time rather than runtime. There are some constructs that we would like to represent with a high level programming language such as Rust or C++, but for which, if we know the parameters to the function at compile time, then we can also calculate the output at compile time as well.

One thing to note is that it's **not** possible to do everything at compile time since we generally require I/O, and whenever I/O is involved it can no longer be performed or baked in at compile time. However we can isolate sections of the program that are able to be executed by the compiler and ensure they can be executed in a constant evaluation environment.

---
# Why is it needed? #

Most of the time, source code is compiled by a compiler into machine code which then executes on the target machine. The compiler performs checks such as type checking, and when it outputs the machine code has additional duties such as filling in function pointers to virtual functions, etc. The source code provides a nice high level way to code using types, interfaces, and other such concepts, and the compiler has the job of turning that into CPU instructions.

However, the more CPU instructions are required, the slower the execution will be. Usually we are trying to target a high frame rate and are therefore limited in the time we can spend on each frame. Every operation performed by the CPU will eat up this budget.

The more calculations we can move from runtime to compile time, the better the execution will be. Additionally since the program is compiled generally much less than it's executed, it would be a better trade-off of time to have a longer compile time but shorter run time.

Another point is that if calculations are done at compile time, with the results stored in the binary itself in the data section, older hardware will be able to run it due to the lower load on the CPU required.

---
# How do we do this in Rust? #

Compared to a language like C++, Rust has comparatively weaker compile time capabilities, but these capabilities are being expanded all the time due to the open source nature of the language. However, compared to other languages that have no compile time/const evaluation capabilities at all, the features we have access to are decent enough.

There are four main building blocks for constant evaluation in Rust. The most common method of metaprogramming is the use of macros.

#### Macros ####

A macro in rust looks like a function, except that instead of taking values as parameters, and producing a value as an output, it takes code as input and produces replacement code as output.

In languages without a deep macro system such as C++, macros are often implemented as simple textual substitutions with no knowledge of the types of syntax accepted.

In Rust there are actually 2 different types of macros:

1. Declarative macros - these look a lot like functions in Rust, but are different, and identified as a macro by the little "!" that Rust requires be placed before the opening parenthesis. These are more powerful than macros in C++ for example, but more limited than the second form.
2. Procedural macros - these are compiled in their own separate library and imported to be used, but these have access to the entire Abstract Syntax Tree (AST) that they're decorating. Whereas declarative macros can only add/expand syntax and can't remove syntax, proc macros can rewrite the entire item they're attached to. These are more similar to function decorators in Python.

There's too much to macros to explain here, so i'll do a separate post on them which will let me explore them deeper myself too. However, for a quick example of a macro, here's one that can create a HashMap for us:

```rust
macro_rules! hashmap {
    // This case handles the logic and is used when the parameter list does NOT
    // have a trailing comma (e.g. hashmap!(22 => "twenty-two", 35 => "thirty-five"))
    ($($key:expr => $value:expr),*) => {
        {
            let mut hm = ::std::collections::HashMap::new();
            $(hm.insert($key, $value);)*
            hm
        }
    };
}

fn main() {
    let hm = hashmap!("one" => 1, "two" => 2, "three" => 3);
}
```

#### Generics ####

Generics allow for complex composite types to be defined that are known in their entirety at compile time, but which the compiler might be able to compile down to a no-op in certain cases. We call these *zero-cost abstractions* and they're quite common in Rust. It's common to define a transparent wrapper around another type, for example:

```rust
#[repr(transparent)]
struct NewType<T>(T); // NewType is a wrapper around T
```

In the above snippet we can make a wrapper type, and we can compose it with other generic types as needed to be able to give information to the compiler for reasoning, and then have that entirely compiled away at runtime so there's no overhead.

The next tool in our compile time toolbox in Rust is the const function

#### Const function ####

Whereas macros allow us to expand or replace syntax, const functions let us write familiar looking (albeit very restricted) functions in a way that let them be executed at compile time if the parameters are constant.

In a macro, the syntax expansion is performed at compile time but the resulting syntax is still only executed at runtime, which allows us to still make use of heap-allocated structures and such.

In a const function, the function itself is executed at compile time (if the parameters are constant), but can still be run at runtime (if the parameters are non-constant).

However because we **must** write the function with the possibility that it's executing at compile time, we have a whole host of things we can't make use of. In addition to not being able to heap allocate structures, we can't for example, make use of trait bounds or traits in a const function.

Due to these limitations, we have to code in a more low level and possibly unsafe way than we have before, but this is something I'll be exploring in the upcoming posts.

As a simple example, the following is a const function that takes two integers and adds them together:

```rust
const fn add_together(a: u32, b: u32) -> u32 {
    a + b
}
```

#### Const keyword ####

The const keyword identifies to the compiler that this is a const context. The left-hand side of a const declaration **must** explicitly state the type (no type inference). The right-hand side of a const declaration **must** be entirely calculable at compile time.

A function written as a const function (as the example add_together function above) can also be invoked at runtime with parameters deduced at runtime, however we can force it to be executed at compile time by introducing a const context as such:

```rust
const ADDED: u32 = add_together(add_together(1, 2), add_together(3, 4));
```

The compiler will execute this at compile time and the resulting binary will simply have the result (10) embedded.

---
# Limitations #

The limitations of writing code to be executable within a constant context are detailed [HERE](https://doc.rust-lang.org/reference/const_eval.html)

However the main limitations are:
1. No heap allocations (no allocator present in the compiler)
2. No floating point operations (floats aren't reliable enough for use in the compiler)
3. No bounds on generics in const functions
4. No comparing raw pointers (pointers don't exist in the same way in the compiler)

The other big limitation will be that large parts of the standard library aren't executable in constant contexts and that we will have to find other means to build the same functionality.

There are crates that have been developed to add this functionality into a constant context but for the sake of education, we'll be developing this kind of functionality ourselves into a package called [consteval](https://github.com/ForgottenMaster/consteval)