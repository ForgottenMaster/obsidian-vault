We now almost have all the pieces in place in the refactored API to be able to train a neural network using the described typestates in the previous sections. The final thing we need to be able to do is to run the forward and backward passes, and apply optimisation to the network weights.

In the last post I covered the Forward trait and described why it needs to be generic over a lifetime, so we have our way of performing the forward pass.

This post then will be a description of the object/typestate representing that forward pass, and the method on which to run the backward pass and optimisation on it.

---
# ForwardOperation #

This trait is a simple one and provides a way to run the backward pass on a previously run forward pass.

The trait is as follows

```rust
pub trait ForwardOperation: Sealed {
    type Output;
    type Input;

    type Backward: BackwardOperation;
    fn backward(self, output_gradient: Self::Output) -> Result<(Self::Backward, Self::Input)>;
}
```

It has an associated type identifying the output gradient type, and an associated type identifying the gradient at the operation's input given that output gradient.

The return type is a result which can either be a structure representing this completed backward pass, along with the actual value of the input gradient. If the output gradient shape is wrong, or due to some other error then the result can be an error.

The associated type Backward is constrained to implement the BackwardOperation trait to allow for functionality from the BackwardOperation trait to be usable in a generic context.

---
# BackwardOperation #

This trait is the simplest one of the lot and is the final typestate in the typestate machine. The trait is defined as such

```rust
pub trait Operation: Sealed {
    fn optimise(self);
}v
```

It has a single method "optimise" which **consumes** the backward pass and applies optimisation to the weights of the network based on the calculated gradients.

Due to this consuming the backward pass, the mutable borrow we took on the trainable network is no longer around and therefore we're free to start a new forward pass for the next iteration.