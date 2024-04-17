# What are traits? #

Traits in Rust can basically be thought of as interfaces in C#. They can do everything that a C# interface can do except with a few more capabilities. We will start off by equating the common functionality of traits in Rust with C# interfaces, and then explore the additional capabilities we get with Rust traits.

---
# Describes capabilities #

In C# interfaces, we can describe a set of function signatures which will tell the user the capabilities of that interface, such that they know when they call something what data to pass in, and what they should get back. The caller need not know how the function is implemented by a specific type, just that it does what is described on the tin.

For example, a C# interface which provides the capability to get an identifier through a GetIdentifier function which takes no arguments, and returns the ID of the instance as an integer can look as follows:

```cs
interface IGetIdentifier
{
    int GetIdentifier();
}
```

In Rust we can describe the exact same functionality. Note however a few changes in order to follow naming conventions or required by the language syntax:

1. interface keyword is renamed to trait
2. In Rust, the naming convention for a trait that has a single method is just the name of the method itself
3. Rust has sized integer types so int is replaced with i32 (can be any of the other integer types too)
4. Rust requires each member function to take explicitly the object we're calling on. This allows us to tell the compiler/caller whether we're taking an immutable reference, mutable reference, or taking ownership. In this case we only need an immutable reference to get an identifier
5. Rust naming convention for functions and variables is snake_case. Therefore GetIdentifier method is renamed get_identifer.

The equivalent Rust definition is therefore:

```rust
trait GetIdentifier {
    fn get_identifier(&self) -> i32;
}
```

---
# Generics #

In C# we are also able to make an interface generic. This is useful if we need to implement an interface multiple times for a given type for different situations. We can't implement the same interface multiple times, but each different set of generic parameters is essentially a different interface.

Let's say we extend the above interface to work for any identifier type and not just integers. The C# snippet would be extended to look as follows:

```cs
interface IGetIdentifier<T> 
{
    T GetIdentifier();
}
```

Likewise, the Rust snippet is extended with the exact same syntax:

```rust
trait GetIdentifier<T> {
    fn get_identifier(&self) -> T;
}
```

---
# Bounded generics #

This is the ability to constrain generic parameter types to ones that only implement certain interfaces. In C# we have the ability to constrain based on interface or base class type, however in Rust we don't have struct inheritance but do have trait inheritance. As a result, in Rust we can only constrain on interface. But this is good practice anyways as inheritance of behaviour is better than inheritance of state.

In C# if we want to bound the above interface to only allow T's that implement an "IIdentifier" interface, that is an interface that allows the type to be used as an identifier, would look like this:

```cs
interface IGetIdentfier<T> where T : IIdentifier 
{
    T GetIdentifier();
}
```

In Rust, we have two options for defining trait bounds, we can do it as above with a "where" syntax. This is useful when there are lots of bounds for a type and we can break them up over multiple lines which is more readable in most cases:

```rust
trait GetIdentifier<T>
    where T: Identifier 
{
    fn get_identifier(&self) -> T;
}
```

However we also have the ability to define these trait bounds inline, which for fewer trait bounds could be neater:

```rust
trait GetIdentifier<T: Identifier> {
    fn get_identifier(&self) -> T;
}
```

---
# Default implementations #

Since C#8 we have the ability to provide default implementations for methods, to be used if the implementor doesn't provide their own implementation. This lets us define some required methods that must be implemented, by not defining a default implementation, and provided methods (which can still be overridden, but don't need to be) by giving a default.

For example in C#, if we assume that the definition of the "IIdentifier" interface is as follows:

```cs
interface Identifier
{
    string GetString();
}
```

Then we can add a provided method to the GetIdentifier interface which will print out the string identifier (note that we still require the GetIdentifier function to be implemented since there's no possible way we can know how to get one). We get the resulting code:

```cs
interface IGetIdentifier<T> where T: IIdentifier 
{
    T GetIdentifier();
    void PrintIdentifier()
    {
        System.Console.WriteLine($"{GetIdentifier().GetString()}");
    }
}
```

We can do the same in Rust too in the same way as the following code snippet shows:

```rust
trait GetIdentifier<T: Identifier> {
    fn get_identifier(&self) -> T;
    fn print_identifier(&self) {
        println!("{}", self.get_identifier().get_string());
    }
}
```

---
# Implementing #

We can of course implement an interface or trait in both C# and Rust (what use would an interface be if we couldn't!?). However the syntax is different, and this is where we start to see C# and Rust diverge quite dramatically.

In C# we have to provide the implementation of the interface at the same point as we define the implementing class/struct itself. For example, implementing the IIdentifier interface for a struct Foo:

```cs
struct Foo : IIdentifier
{
    public int identifier;
    public string GetString() => $"{identifier}";
}
```

And implementing the IGetIdentifier\<Foo> trait for a second struct FooGetter:

```cs
struct FooGetter : IGetIdentifier<Foo> 
{
    public Foo identifier;
    public Foo GetIdentifier() => identifier;
}
```

In Rust however, the difference is that we define implementation blocks separate to the variables inside the struct itself. This allows us to break up behaviour/functions from the pure data contained in the structure. Each trait has its own implementation block so the implementation of the above structures and trait implementations will look as follows:

```rust
#[derive(Clone)]
struct Foo {
    pub i32 identifier;
}

impl Identifier for Foo {
    pub fn get_string(&self) -> String {
        format!("{}", self.identifier)
    }
}

struct FooGetter {
    pub Foo identifier;
}

impl GetIdentifier<Foo> for FooGetter {
    pub fn get_identifier(&self) -> Foo {
        self.identifier.clone()
    }
}
```

Note the Clone implementation required on Foo, and the clone function call in get_identifier. This is required because in Rust every type is movable, and clones are explicit. Adding the #[derive(Clone)] attribute to the struct allows us to automatically derive a deep clone implementation as long as all the struct fields implement Clone.

Note that this is **not** idiomatic Rust, clones are rarely used as we can pass references around safely and the compiler will check the ownership and lifetime rules for us.

---

# Is that it? #

So you might look at the previous content and think that Rust traits **are** just C# interfaces with a different syntax. They can do everything C# interfaces can do right?. Well, yes, except that there are more capabilities that Rust gives us that just aren't possible in C#

The following few sections then will only contain Rust snippets as there's no valid way to represent them in C# (or C++, or most languages I've used - except Haskell, which makes traits similar to typeclasses).

---

# Associated Types #

In the above code snippets, we had a generic trait GetIdentifier which was implemented on FooGetter with the generic parameter Foo. However this opens the way for us to have multiple implementations on FooGetter with different types. However what if we want to force the user to define a maximum of 1 implementation of GetIdentifier?

Well, we have to remove the generics, and we end up with a trait as follows:

```rust
trait GetIdentifier {
    fn get_identifier(&self) -> ??? // what type goes here?
}
```

However, we have a problem here since we don't know what the actual identifier type is now for any given implementation. We've removed the ability to specify it in a generic parameter, so in C# the only way to do this would be to fix the concrete type we return. That means all implementors of GetIdentifier returns the same type.

Technically in C# we can do it by returning the IIdenfier interface itself:

```cs
interface IGetIdentifier 
{
    IIdentifier GetIdentifier();
}
```

Which indeed will allow each implementor to determine what the actual type they're returning is, as long as it implements the IIdenfier interface.

The downside here is that we are forced to box the result which means a heap allocation, which means garbage collector tracking overhead.

We can do the same thing in Rust, we have to be explicit about returning a dynamic trait object in a box though:

```rust
trait GetIdentifier {
    fn get_identifier(&self) -> Box<dyn Identifier>;
}
```

However this still requires allocating heap storage and returning. An additional downside is type erasure. We've lost all information about the actual concrete type, all we know is the Box has an Identifier in it, so can only access the methods of the Identifier trait and nothing more.

There must be a better way!?. Enter associated types:

```rust
trait GetIdentifier {
    type IdentifierType: Identifier;
    fn get_identifier(&self) -> Self::IdentifierType;
    fn print_identifier(&self) {
        println!("{}", self.get_identifier().get_string());
    }
}
```

Problems solved!. There's a bit of new syntax here, but the main points are:

1. We define an associated type on the trait with the **type T** syntax, allowing each implementor to specify a different **concrete** type
2. We place a trait bound on it so that only concrete types implementing the Identifier trait can be used
3. We return Self::IdentifierType from the get_identifier function. Importantly this is the **concrete** type, meaning no heap allocations and no type erasure

One further thing with associated types is that we can add trait bounds that force them to a specific concrete type. For example, say that we want to create a function that will take any GetIdentifier and print it, but only if the identifier type is an i32. We can do this as a trait bound with the following syntax:

```rust
fn call_print_identifier_if_i32<T: GetIdentifier<IdentifierType = i32>>(t: &T) {
    t.print_identifier();
}
```

We can also even place trait bounds on associated types within trait bounds!. For example if we want a function that will accept any GetIdentifier, but only if the IdentifierType implements Clone, we can do so:

```rust
fn do_something<T>(t: &T) 
where 
    T: GetIdentifier,
    <T as GetIdentifier>::IdentifierType: Clone
{
    let c = t.get_identifier().clone();
    // do something with cloned instance
}
```

---

# Associated methods #

Unlike C# (and most other languages) interfaces which can only define methods tied to the specific instance of the implementing type, due to requiring dynamic dispatch and using a vtable, in Rust we can also define methods at the type level. These would be called static methods in other languages but in Rust, they are known as associated methods.

A simple example of a trait making use of this functionality is the Default trait in the standard library. If we were to define it ourselves, we can do it like this:

```rust
trait Default {
    fn default() -> Self;
}
```

Self here is the implementing type, and notice how an associated method is indicated not by a keyword, but by the lack of self, &self, or &mut self in the first argument position.

These associated methods can be called with :: syntax, for example if our type Foo implements Default, we can call it as:

```rust
let default_foo = Foo::default();
```

---

# Extension traits #

The next feature Rust provides us with respect to traits is as a side effect of having to implement them separate to the type we're implementing on. This implies we can implement traits for types that we don't actually own. This isn't possible in C# or C++ where the implementation of a type is defined at the same time as the type itself.

This means that we are able to add functionality to existing types, even standard library types or primitive types by creating a trait and implementing it.

For example, say that we want to add a AsBytes trait which specifies that the type has a method called as_bytes which returns a vector of u8's representing the bytes of the type.

Such a trait can be defined like:

```rust
trait AsBytes {
    fn as_bytes(&self) -> Vec<u8>;
}
```

And we can implement that for our own types, however, unlike in most other languages, we can implement this for existing types even primitives. For example on a u32:

```rust
impl AsBytes for u32 {
    fn as_bytes(&self) -> Vec<u8> {
        self.to_le_bytes().to_vec()
    }
}
```

to_le_bytes is a function that the standard library provides for us, that will give us an array of u8's of length 4 with the bytes of the u32 in it. We can call to_vec on this to turn it into a dynamically sized vector instead to return.

---

# Blanket implementations #

The final feature for traits that we have with Rust is the ability to implement a trait for all types, optionally bounded with trait bounds.

Let's say we want to add a method to **all** iterators which will result in a new iterator that prints out the item (for all items that are displayable) as it iterates them.

Note that we can do this with a map call to decorate a function, taking the input, printing it and returning it again to make a new iterator. This would look something like this:

```rust
let iter = (0..10).map(|elem| {
    println!("{}", elem);
    elem
});
```

However this requires the user to roll the function themselves to print and return, and is a bit unwieldy. What we'd like is:

```rust
let iter = (0..).print();
```

We can do this with a blanket implementation that adds this print function to all iterators. First we need a structure to wrap the iterator that will step through and do the printing:

```rust
struct Print<T> {
    iterator: T
}
```

We will need to implement the Iterator trait here for the new structure, so that we can step over and print the elements. However we can only do this if the elements implement the Display trait. We can use trait bounds to ensure that:

1. T is an Iterator
2. The elements from T implement Display

This will look as follows:

```rust
impl<T: Iterator> Iterator for Print<T>
where
    <T as Iterator>::Item: Display
{
    type Item = <T as Iterator>::Item; // just passing the items through
    fn next(&mut self) -> Option<Self::Item> {
        let item = self.iterator.next()?;
        println!("{}", item);
        Some(item)
    }
}
```

The little ? syntax when we call the next function of the iterator we're wrapping is a little outside of the scope of the article, but the easiest way to think of it is if that call returns None, then we return None immediately. Otherwise item is set to the value inside the Some (which we then print and return).

Finally we need to actually add the convenience function to all iterators. We can do this by creating an extension trait called IteratorPrint with the print function we want:

```rust
trait IteratorPrint
where 
    Self: Sized 
{
    fn print(self) -> Print<Self> {
        Print {
            iterator: self
        }
    }
}
```

We need to specify Self: Sized because traits in Rust can be implemented even for dynamically sized types which can't exist on their own and must be boxed or put behind a reference.

Since we need to put self into the Print structure, it will need to have a compile time known size, so we specify we can only use it with such types.

Now we have all the boilerplate setup, we can actually do the blanket implementation. We'll add it only to compatible iterators, otherwise we end up with a more cryptic compile error. The actual final code for this is as follows:

```rust
impl<T: Iterator> IteratorPrint for T
where
    <T as Iterator>::Item: Display
{
}
```

Then we can take **any** iterator that has elements that are displayable, and use this print function on it, even if we didn't write the iterator type ourselves!. For example the following is valid:

```rust
(0..10).print().for_each(|_| {});
```

The for_each call will just apply the closure to each element, in our case we just want to do nothing, but it will trigger the prints as it iterates.
