After building the network/operation chain and initialising it with weights/parameters either via a known set (through previous training), or via a random seed (using Xavier initialisation) we have a network that is capable of making predictions given some input. We are also at this point able to read out the set of parameters for later use.

We are only able to get an operation into the initialised state by first starting with an uninitialized version and then initialising it, we can't directly construct new instances of this type because it's not publicly visible/nameable.

Thirdly we're able to take this initialised operation and put it into "training" mode by binding an optimiser with it.

---
# The InitialisedOperation trait #

First we'll take a look at the overall trait definition:

```rust
pub trait InitialisedOperation: Sealed {
    type Input;
    type Output;
    type ParameterIter: Iterator<Item = ElementType>;

    fn iter(&self) -> Self::ParameterIter;
    fn predict(&self, input: Self::Input) -> Result<Self::Output>;
}
```

As before, and as with all the traits in Eidetic, this is a Sealed trait to allow us to control implementation details and correctness by implementing it only for operations inside the crate. In fact, it's likely this won't be pointed out again on any future traits.

As far as associated types go, we need:

1. **Input**. The type of input to the operation, always a tensor type.
2. **Output**. The type of output from the operation, again a tensor type.
3. **ParameterIter**. The type of iterator that is produced from the iter method to gain access to the stream of trained weights.

The parameter iterator will produce elements/weights in the same order as they are expected when initialising a network of the exact same architecture via the with_iter method, so this can be used to store those weights off to a file or transmit across a network, or otherwise store for later use.

The other capability is the ability to make predictions for the operation via the **predict** method. This will take an input of the appropriate type, and produce an output. Note that this is a *fallible* method because the input may not be correctly shaped at runtime for the operation (e.g. an incorrect number of columns/features), so it returns a Result type which can either be propagated or unwrapped as needed.

---
# What's the next step? #

You may have noticed that this trait does not have any way to move the type state over to the next state (the trainable state). This is because the output type will depend on the specific optimiser chosen to use to put this operation into a trainable state.

This means the trainable state is generic over the optimiser in use, since it needs to hold onto an instance of an optimiser to support adaptive optimisation and things like learning rate decay.

Since each type can only have one implementation of a given trait, and since we want the initialised operation to be moved to trainable with any compatible optimiser, we need a separate trait that itself is generic over the optimiser type.

---
# The WithOptimiser trait #

This secondary trait is the WithOptimiser trait which looks as follows:

```rust
pub trait WithOptimiser<T>: Sealed {
    type Trainable;

    fn with_optimiser(self, optimiser: T) -> Self::Trainable;
}
```

T in this case is the optimiser type we're using (actually, we'll see later it's an optimiser *factory* because it needs to support *making* an optimiser that's generic over a particular type). This lets us implement versions of this trait for each optimiser type in existance which lets the caller pick the optimiser to use.

The Trainable type here is the type which is used for this operation that is bound to an instance of that optimiser. For operations that don't have any parameters and thus, no need for optimisation, they will always produce the same type to represent their trainable form regardless of the optimiser chosen.

For types that do require storing an optimiser for later use during the optimisation stage, the correct implementation can be chosen here.

Rust will pick the correct version of the trait to use when the with_optimiser function is called because it will be able to see what the type of T is, due to it requiring an instance into the with_optimiser function. Therefore all of the following calls will be correct:

```rust
let null = initialised.with_optimiser(NullOptimiser::new());
let sgd = initialised.with_optimiser(SGD::new(FixedLearningRateHandler::new(0.01)));
let sgd_momentum = initialised.with_optimiser(SGDMomentum::new(FixedLearningRateHandler::new(0.01), 0.9));
```

---
# Why the optimiser isn't an optimiser? #

As mentioned, the type T that is passed into the with_optimiser function (e.g. NullOptimiser, SGD, etc.) is not implementing the Optimiser trait directly but another trait called OptimiserFactory. We'll look into more details when we cover the optimisation stage, but the reason for this is because an optimiser type will require storing state that is dependant on the type of parameter for a given optimisation (e.g. Tensor of a certain rank).

However, for operation chains/composite operations we want to be able to take the type T and pass it to both/all operations in the chain - but they might store different parameter types. Therefore the type with_optimiser takes is a factory which can produce optimiser instances of various types when required by those operations.