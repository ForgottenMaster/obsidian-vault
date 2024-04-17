# C Style Enums #

In C#, C++, and a lot of other popular programming languages, we have access to a type called an "enumeration" (or enum for short). This is simply a type safe collection of named constant values.

For example in C++, making an enum whose variants represent a set of allowed colors for a hypothetical UI framework could be written as (with the values of the variants explicitly typed out for transparency):

```cpp
enum Color {
    Red = 1,
    Green = 2,
    Blue = 3
};
```

A function can then go ahead and accept a "Color" and the user will be able to pass only the named colors. Except that in C++, this is not true. Nothing stops the caller from casting an arbitrary integer as a "Color". For example calling a SetColor function that takes a Color, the caller can do:

```cpp
widget.SetColor(static_cast<Color>(10)); // what even is color with value 10???. It hasn't been defined so likely won't be correctly handled.
```

This is not desired as SetColor can't assume that the given Color is only one that was specified in the enumeration. If someone can arbitrarily cast an integer to Color, what is the function supposed to do with it?.

In Rust, we can have value type enums the same way:

```rust
enum Color {
    Red = 1,
    Green = 2,
    Blue = 3
}
```

And casting such an enum value into an integer is totally fine, the following snippet will print "Selected color is: 3":

```rust
// enum variant to integral value is safely supported because it's a total function - all enum variants in this style of enumeration
// can be cast to the respective integer value.
println!("Selected color is: {}", Color::Blue as u8);
```

This is totally fine and allowed by Rust because all variants of this C-style enumeration can be casted safely to an integer. We say it's *infallible*

However, casting an integer to an enumeration type is not infallible because not all possible integral values have a variant in the enumeration, we can't do the following - it simply does not compile:

```rust
// does not compile as integral to enum conversion is not implemented since it can fail for certain values of integer.
let int_as_color = 10 as Blue;
```

Rust is a safe and cautious language and the compiler will just not allow operations that could fail, therefore converting from enum to integral isn't supported by default, however the enum creator can implement the TryFrom<T> trait for any integral types.

However this is boilerplate that's already been done and available in a crate. Therefore to allow for C-style enums in Rust with safe conversions in both directions, it's best to use [num_enum](https://crates.io/crates/num_enum)

---

# Tuple Enums #

In C++, the above is all you get, loosely typed integers that aren't even that safe. With Rust, enums become more powerful with the ability to store different data inside of each variant.

These are similar to an algebraic data type such as Haskell has. It could be thought of similar to a union in C++ in that an element of the enum type takes up the amount of space required for the biggest variant (allowing it to store in an array), but strongly typed so you can't access data you shouldn't.

As an example, suppose we want to have an "Angle" enumeration. An angle could be stored in either Degrees or Radians, but both are stored as floats. In this case the tuple enum would look as follows:

```rust
enum Angle {
    Degrees(f32),
    Radians(f32)
}
```

In both of these variants, we store an f32, but behind the scenes each instance of Angle is tagged with its discriminant (Degrees or Radians) and the only way to access the data inside is through pattern matching. This means we literally can't access data that we shouldn't for the variant we have. An example of implementing the Into<f32> trait for this would be:

```rust
impl Into<f32> for Angle {
    fn into(self) -> f32 {
        match self {
            Self::Degrees(val) => val,
            Self::Radians(val) => val
        }
    }
}
```

An example of an enumeration with differing types could be a Color, where we can choose between different color formats:

```rust
enum Color {
    RGBF32(f32, f32, f32),
    RGBAF32(f32, f32, f32, f32),
    RGBU8(u8, u8, u8),
    RGBAU8(u8, u8, u8, u8)
}
```

That is, we can choose between colors with RGB or RGBA components, and can choose the type of the components we're storing. However because this is essentially a strongly typed union, and Rust requires all types to have a defined size to be stored, the largest size would still be picked, in this case each Color would be 16 bytes large (corresponding to the size of RGBAF32 which is largest).

Pattern matching works the exact same way. For example, a method on Color which can return whether the color supports transparency could be written as follows:

```rust
impl Color {
    fn supports_transparency(&self) -> bool {
        match self {
            Self::RGBF32(..) => false,
            Self::RGBAF32(..) => true,
            Self::RGBU8(..) => false,
            Self::RGBAU8(..) => true
            
        }
    }
}
```

---

# Named Field Enums #

A variant in an enum can use the tuple syntax for defining the types it contains, and pattern matching as described above, however we can also store values associated with an enum variant by name in a record/struct like syntax. We can freely mix and match these on a per-variant basis. For example an enumeration which represents an Error. We might support storing an ErrorMessage, ErrorCode, or both. This might look as follows:

```rust
enum Error {
    Message(String),
    Code(i32),
    Both {
        message: String,
        code: i32
    }
}
```

When pattern matching on tuple types, we need to use the tuple patterns. When matching on record types, we need to use that syntax. An example of a function to try to get an error code from an Error would be:

```rust
impl Error {
    fn try_get_code(&self) -> Option<i32> {
        match self {
            Self::Message(..) => None,
            Self::Code(c) => Some(*c),
            Self::Both{code: c, ..} => Some(*c)
        }
    }
}
```

---

# Empty Enums!? #

Empty enums can't be constructed. This may sound kind of pointless, what does

```rust
enum Void {
}
```

Even mean if it can't be constructed?

As it turns out this can be very useful for statically proving that we can't ever take a particular branch of code in some cases, and is sometimes seen in generic code.

For example, in Rust we have the Result type which has two type parameters. One is the success type, and one is the error type. Say that a trait requires a return type of Result<SuccessType, ErrorType> from a function

```rust
trait TryOperation {
    type SuccessType;
    type ErrorType;
    fn try_operation(&mut self) -> Result<Self::SuccessType, Self::ErrorType>
}
```

In order to implement such a trait, we **must** provide an ErrorType to satisfy the signature of the trait, but for an infallible operation which is guaranteed to not fail, what do we choose for an ErrorType?. Any type is as good as any other type if we never need to construct it and we never return it:

```rust
struct InfallibleOperation {
}

impl TryOperation for InfallibleOperation {
    type SuccessType = (); // Unit type is like "void" in other languages, it's a type we can use when we don't need to return any information.
    type ErrorType = ???; // What do we put here if this operation never fails?
    fn try_operation(&mut self) -> Result<Self::SuccessType, Self::ErrorType> {
        println!("Hello, World!");
    }
}
```

The implementation of this function simply prints to the console and never fails, we're "trying" to perform the operation but it will always succeed. It turns out in these cases we can communicate this at the type level to the caller by using the empty enum as the ErrorType (e.g. Void). If the caller sees that the signature for this is:

```rust
fn try_operation(&mut self) -> Result<(), Void>
```

Then they can easily see that the error case can **never** possibly happen (because an instance of Void physically cannot be constructed). In this case, the caller knows it's safe to not even handle the possibility of error as this is ensured by the compiler in the types.
