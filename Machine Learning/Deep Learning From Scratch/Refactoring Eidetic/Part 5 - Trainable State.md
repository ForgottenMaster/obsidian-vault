Now we've bound a particular optimiser to the operation(s) it is placed into the trainable state. The optimiser instances will retain additional state for gradients, learning rates, etc. that aren't needed in the previous initialised state.

In this typestate we are able to run training passes (as opposed to just predictions) on the network and also, once finished, to go back to the initialised state again from which we can get the trained parameters.

---
# TrainableOperation #

This trait is pretty straightforward actually. Let's take a look at the definition in it's entirety for a network/operation that is ready to be trained:

```rust
pub trait TrainableOperation: Sealed {
    type Initialised;

    fn into_initialised(self) -> Self::Initialised;
    fn init(&mut self, epochs: u16);
    fn end_epoch(&mut self);
}
```

There is only a single associated type with this one which is the type representing the previous state of operation (the initialised state). For functionality we have, as previously mentioned the two pieces of functionality we would like to use.

---
# During training #

For the actual training process we will need to initialise all the optimisers in the trainable network with the number of epochs that we're running over. This is because those optimisers will use things such as learning rate decay, or otherwise be reliant on the total number of epochs we are running. Initialisation also lets them know we're about to train in epoch 0, so can reset any data that needs to be (such as resetting those decaying learning rates back to starting values).

At the end of each epoch, the trainer will then call **end_epoch** to give any optimisers and such in the network the chance to update state before the next epoch (such as decaying learning rates).

---
# After training #

After we have finished running our training for however many epochs we'd like, we then need to be able to go back to the previous state in order to extract the trained weights for storage, or to make predictions without the overhead of the optimiser instances.

The **into_initialised** method will do this for us and it's fairly straightforward. It simply *consumes* this trainable instance and produces an initialised form. In effect, this *unwraps* the trainable operation and strips away any additional data used during the training process (e.g. gradient velocities for momentum) and the resulting operation is lean, only containing what it needs to make predictions.

---
# Where's the forward pass? #

As you *may* have noticed, the trait defining an operation as being in the trainable state doesn't actually have any functionality to run a forward pass (and thus to run a backward pass, etc.).

The reason for this is simple, we need to put that functionality into a *separate* trait, muchlike we did with the binding of the optimiser and it's for the same reason - generics. However the generic parameters here are not generic types, but **lifetimes**.

---
# A consuming transition #

Let's first take a moment to look at what the API would be if the typestate transition from "trainable" to "forward pass" consumed self in the same way as the transitions so far do:

```rust
let mut trainable = get_trainable_network();

// run one epoch...we'll pretend we know the output gradients before we've run our
// forward pass for brevity.
trainable = trainable.forward(input).backward(output_gradients).optimise();
```

As you can see, the API is a little....*clumsy* since you have to remember to re-assign to the trainable binding after running an epoch, since each step consumes self. If we didn't re-assign and instead did something like:

```rust
let mut trainable = get_trainable_network();

// Epoch 0
trainable.forward(input).backward(output_gradient).optimise();

// Epoch 1
trainable.forward(input).backward(output_gradient).optimise(); // COMPILER ERROR! trainable already used
```

---
# A non-consuming transition #

Ideally, what we would like to do is to have the forward transition be non-consuming and only **borrow** the operation for its duration. We would like our API to look more like this:

```rust
let mut trainable = get_trainable_network();
trainable.forward(input_0).backward(output_gradient_0).optimise(); // Epoch 0
trainable.forward(input_1).backward(output_gradient_1).optimise(); // Epoch 1
// etc., more epochs
```

The problem here of course is that we're only *borrowing* trainable for each forward pass, and the borrow will last until it's been finished with. Rust doesn't allow multiple *mutable* borrows in existence at the same time because if it did, we could theoretically write something like this:

```rust
let mut trainable = get_trainable_network();
let epoch_0 = trainable.forward(input_0).backward(output_gradient_0); // calculate gradients to update parameters with for epoch 0 but don't apply
let epoch_1 = trainable.forward(input_1).backward(output_gradient_1); // calculate gradients to update parameters with, but from the same parameters as epoch 0.
epoch_0.optimise(); // apply gradients to parameters
epoch_1.optimise(); // apply gradients to different parameters used to calculate them!
```

So to avoid this, we take out a *mutable* borrow for the duration of one epoch which means that we always start the next training pass from a state that we are in sole control over changing - the previous pass has either been dropped or applied.

This means we need to *store* a mutable borrow in the struct as we go through the typestates, and storing a mutable borrow in a struct requires an explicit lifetime.

---
# Why can't we add forward to TrainableOperation? #

At first you might try to add the forward operation to the trainable trait, which we'd think would look something like this:

```rust
pub trait TrainableOperation : Sealed {
    .. // previous stuff we've covered

    type Forward;
    fn forward(&mut self) -> Self::Forward;
}
```

However, as mentioned the type of Self::Forward will require an explicit lifetime. The lifetime in this case will be the lifetime of the self borrow to forward. This implies that Forward itself needs to be generic over the lifetime, something like:

```rust
pub trait TrainableOperation : Sealed {
    .. // previous stuff we've covered

    type Forward<'a>;
    fn forward<'a>(&'a mut self) -> Self::Forward<'a>;
}
```

That is, when we define the associated type Forward, we define it as having a single generic parameter (the lifetime 'a), and in the forward method we set that lifetime to be the same one that &mut self has.

This, unfortunately, is not possible at the time of writing in Rust as it requires a feature called **[Generic Associated Types](https://rust-lang.github.io/rfcs/1598-generic_associated_types.html)** which is still being worked on.

---
# Forward trait #

So, as mentioned before we can do this by putting our forward method into a trait which is *itself* generic over that lifetime. We can then implement this trait for all lifetimes and when we call "forward" on an implementor, Rust can determine which trait instance should be used.

The trait then looks as such:

```rust
pub trait Forward<'a>: Sealed + TrainableOperation {
    type Input;
    type Output;
    type Forward;

    fn forward(&'a mut self, input: Self::Input) -> Result<(Self::Forward, Self::Output)>;
}
```

I've put an additional bound here to make sure we only implement Forward on those operations that are in the TrainableOperation state. The trait as mentioned is generic over a single lifetime, 'a, which we then use to explicitly say this is the lifetime of the **&mut self** borrow.

For the function itself, we take in an input and run the forward pass, the result of this will be a tuple where we get access to the output (so we can calculate loss), along with the operation in a state ready to begin the backward pass.

The tuple is wrapped in Result because this is a fallible function, because the input provided, as with making predictions, might be incorrectly shaped for the number of neurons expected.