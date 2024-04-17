Now that we've covered tensors which are the representation of data that is flowing through the API, we can start taking a look at the operations and how we are using typestates to ensure valid usage of the API without unnecessary bloat in code by keeping members around that aren't necessary.

---
# What are typestates anyway? #

If you aren't used to programming with generics at the type level, you may not have encountered type states before, so a quick explanation of what they are might be necessary. Let's first think about runtime representation of states which is commonly seen when using a **Finite State Machine** (FSM).

In programming, using an FSM allows us to ensure that certain operations are invalid at runtime and that we can only transition from one state of value to certain other ones via specific events.

For example an FSM to make a coffee will look as follows:

![Coffee Making FSM](FSM.jpg)

Just as states are a way to encode the current state and valid transitions of **values** at *runtime*, typestates are a way to encode the current state and valid transitions of **types** at *compile time*.

The advantage of typestates are that they don't allow invalid code to even compile. For example in the above FSM, the starting state of the above FSM could be represented by a type and provides methods to transition to the "Grind Coffee" and "Fry Eggs" states.

This could be represented in Rust as:

```rust
struct Start;

impl Start {
    pub fn make_food_and_drink(self) -> (GrindCoffee, FryEggs) {
        ...
    }
}
```

There are two things we need to ensure for typestates to work for us:

1. There should be **no other way** to construct GrindCoffee or FryEggs other than going through the Start::make_food_and_drink method. This ensures that the caller must go through the appropriate flow of actions, starting from the beginning to even construct these types.
2. When we call make_food_and_drink to progress the state machine to the new types, we should **not** be able to use that instance of Start again. This is where Rust's ability to prevent use after move comes in useful.

---
# Typestates in Eidetic #

Now that we have an idea of what typestates are and how they're useful, we can take a look at how we use them in Eidetic. All operations will implement these traits and have ways to represent the various states at the type level, which means we'll only be looking at these typestates at the **trait** level.

As a high level overview the typestate machine will look as follows:

![Typestate Machine](Typestate%20Machine.jpg)

This post and the following posts will run over the traits representing each of these states and describe their API.

---
# Uninitialised #

The very first typestate we have is the Uninitialised one. This is the state that the network/operation is in at the point where the architecture is defined. We can't modify a neural network after it's initialised so we only allow chaining of operations that are in this typestate.

Taking a look at the definition of the trait as a whole:

```rust
pub trait UninitialisedOperation: Sealed + Sized {
    type Initialised: InitialisedOperation;

    fn with_iter(self, mut iter: impl Iterator<Item = ElementType>) -> Result<Self::Initialised>;
    fn with_seed(self, seed: u64) -> Self::Initialised;
}
```

It is intended that any operation or layer that is in the public API is in the uninitialised state, which we can freely connect together to get the desired architecture and then initialise the network as a whole using the provided initialisation methods.

The first thing to notice is that the trait here is Sealed because there are a few additional methods in this trait that are hidden from the public API/documentation that we don't have to want to worry about breaking in external code. A secondary reason to make this Sealed is so that we can ensure that all operations behave correctly according to the contracts defined, and limiting the scope of implementations to this crate only ensures we don't have to worry about breaking changes so much. Thirdly we're hiding away implementation details of our Tensors so there's no optimal way for external code to manipulate tensors to even define their own operations (they would have to convert the tensor to an iterator, then manipulate it, then convert back).

The trait defines one associated type which is the type of the next state, the initialised state. We constrain this to be of a type implementing InitialisedOperation since this knowledge is used for chaining later.

Let's take a closer look at the two ways that the uninitialized network/operation can be initialised:

#### With a seed ####

The easiest method to initialise a network is via a random seed value. This is used when we are initialising the network for the very first time before we've done any training on it to determine optimal weights.

The with_seed method simply takes a u64 which defines the seed to use, and internally initialises any weights using this seed. The return type of this method indicates that it's infallible, and this is indeed correct.

The only way initialisation can fail is if there aren't enough values in a given stream to initialise all parameters/weights in the network, which can only be the case for a finite stream.

With a random seed though, we are generating an **infinite** stream of random values to initialise with, which can't possibly be exhausted. As a result, initialising a network/operation with a random seed can never fail.

#### With an iterator ####
The second method is to initialise the network with a set of weights/parameters that have been determined via training in a previous run potentially.

This method will be useful in the case that we trained the network previously and stored out the trained parameter values to a file and are reconstructing the same network.

In this case, we take an iterator over elements of ElementType (f64 by default) and *try* to initialise the network. Unlike initialising with a seed, this method **can** fail because the provided iterator may not yield enough values to fully initialise the operation.

Therefore this will return a Result containing (on success), the initialised instance.

---
# Conclusion #

Both methods of initialisation result in yielding an initialised version of the operation/network if successful, and more importantly one of these methods is the **only** way for client code to construct one of these initialised instances.

The second thing to point out is that both methods are taking self by *value* which, thanks to Rust's move semantics ensures that the uninitialized version can't be used after it's been initialised. This, in turn allows optimisations because the initialised version can steal the guts of the uninitialized version if needed (for example stealing a pointer or some memory or something).