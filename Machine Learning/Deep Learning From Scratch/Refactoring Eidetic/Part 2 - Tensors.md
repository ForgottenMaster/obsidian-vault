A "tensor" is just a fancy name for "n-dimensional array" (AKA ndarray) and provides a way of thinking about scalars, vectors, matrices, etc. in a uniform way. We refactor Eidetic to use this concept because we want operations to be able to specify how many dimensions are in the input and output data. This post will provide a brief overview of what a tensor is, and how we implement it in Eidetic.

---
# What is a tensor? #

As mentioned in the introduction blurb, a tensor is just simply an array with N dimensions. In tensor terminology the N here is known as it's **rank**. For ease of understanding, there's a few ranks of tensor below with common terminology for that particular data type.

- **Rank 0** => "Scalar". A rank 0 tensor has no dimensions and is a single point. In terms of elements this means it only has a single element contained within which is known as a scalar in mathematics.
- **Rank 1** => "Vector". A rank 1 tensor has only a single dimension which represents its length. All elements are on this 1 dimension and the overall data structure is commonly referred to as a vector (list of things).
- **Rank 2** => "Matrix". A rank 2 tensor has two dimensions which represent the width and height (alternatively rows and columns). Such a tensor is commonly referred to as a matrix.

Higher ranked tensors exist but don't have common names for them and are harder to think about or draw in a diagram. A rank 3 tensor can be thought of as a "cube" or a "stack" of matrices stacked on top of one another and is commonly used to represent the layers in an image for example.

After this they start to become more abstract - a rank 4 tensor can be thought of as a "vector of cubes" perhaps.

---
# Why we need a tensor? #

We need a tensor type in the API because we want to constrain operations and layers to using the same type but allow them to be generic over the rank of the tensor. Using a common type allows us to avoid having to have separate types for scalars, vectors, matrices, etc.

---
# The Tensor type #

Now we have a brief explanation of what tensors are, we can look at how they're implemented inside Eidetic. The following are points that we're trying to satisfy with our implementation:

1. Tensors must be generic over a rank indicating its dimensionality.
2. Tensors must hide all implementation details of how they're stored internally.
3. Tensors should be constructible through a new function which may or may not be fallible.
4. The data inside a Tensor should be iterable.

The type that we settled upon for a Tensor is as follows:

```rust
pub struct Tensor<R: Rank>(pub(crate) Array<ElementType, R::Internal>);
```

This is generic over a parameter type R which **MUST** implement the Rank trait. This is because we need to access the Internal associated type which allows us to define the ndarray::Array type we're using to store the data internally.

This internal state is hidden from the public API by the pub(crate) visibility modifier which means that the public API has no reference at all to the ndarray crate or the Array type. We would easily be able to swap out the implementation for something else as long as the public API continues to use the Tensor type.

pub(crate) gives the crate the ability to work with the ndarray::Array type stored inside however which allows us to internally use all the methods provided by the ndarray crate for calculations while leaving it an implementation detail.

#### Constructing a tensor type ####

In order to construct the tensor type in Eidetic we need to provide a "new" function for each rank of tensor we're constructing. This is due to the parameters and return type being different depending on the rank of the tensor we're constructing (since some ranks are infallible).

For rank 0 tensors (scalars) they will take as input a single element, and return a Tensor - this is infallible because we can always create a tensor with a single element from a single element. The code for this look as such:

```rust
impl Tensor<rank::Zero> {
    pub fn new(elem: ElementType) -> Self {
        Self(arr0(elem))
    }
}
```

For rank 1 tensors (vectors) they will take some type that we can get an iterator of elements from, and will put all those elements into a single dimension in the tensor (it's length). This is infallible because we are simply taking the number of elements in the iterable as the length of the tensor:

```rust
impl Tensor<rank::One> {
    pub fn new(iter: impl IntoIterator<Item = ElementType>) -> Self {
        Self(Array::from_iter(iter))
    }
}
```

Rank 2 and higher tensors are not infallible because this is where we have to take a flat iterator of elements, and reshape it to fit a desired shape. If there aren't enough elements inside the iterator for the requested shape then this will be an error.

I'll only show one implementation here, but they will all follow the same format:

1. Take the iterator provided and make a 1-dimensional array from it
2. Reshape the array into an n-dimensional one using the provided/requested shape
3. If this is an error from ndarray (invalid number of elements) then map the error type to our own Eidetic error type

The code for example for a rank 2 tensor looks like this:

```rust
impl Tensor<rank::Two> {
    pub fn new(shape: (usize, usize), iter: impl IntoIterator<Item = ElementType>) -> Result<Self> {
        let array: Array<ElementType, Ix1> = Array::from_iter(iter);
        let array: Array<ElementType, Ix2> = array.into_shape(shape).map_err(|_| Error(()))?;
        Ok(Self(array))
    }
}
```

Note that we're using the eidetic::Result type alias here since all results output by Eidetic will use the error type of eidetic::Error.

#### Reading from a tensor type ####

Now that we have the ability to construct a tensor from an iterator, we need the ability to go the other way and iterate the values in a tensor which will allow us to interpret the output data from Eidetic as required.

In order to do this we will need to implement the IntoIterator trait provided by Rust, which will iterate over elements of the type eidetic::ElementType (f64 by default, f32 if Cargo feature is enabled). The definition of this trait looks like follows:

```rust
impl<R: Rank> IntoIterator for Tensor<R> {
    type Item = ElementType;
    type IntoIter = TensorIterator<R>;

    fn into_iter(self) -> Self::IntoIter {
        TensorIterator(self.0.into_iter())
    }
}
```

The iterator type is a new type TensorIterator which simply wraps our ndarray::Array type's iterator and delegates the next method. Taking the struct definition and the Iterator trait implementation together we get:

```rust
pub struct TensorIterator<R: Rank>(<Array<ElementType, R::Internal> as IntoIterator>::IntoIter);

impl<R: Rank> Iterator for TensorIterator<R> {
    type Item = ElementType;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.next()
    }
}
```

Again, the wrapped iterator from ndarray is kept private so it doesn't leak into the public API.

---
# The Rank type #

The other part of the Tensor type is the Rank types which are required for 2 reasons:

1. A type for Tensors to be generic over indicating their dimensionality/rank
2. A way to access the associated ndarray type indicating dimensionality as an implementation detail (e.g. Ix0, Ix1, Ix2, etc.)

We use a trait so that we can apply it as a trait bound for tensors. However we make sure to hide the internal type from the public documentation:

```rust
pub trait Rank: Clone + Sealed {
    #[doc(hidden)]
    type Internal: Dimension;
}
```

We are using the [Sealed Traits](https://rust-lang.github.io/api-guidelines/future-proofing.html#sealed-traits-protect-against-downstream-implementations-c-sealed) pattern here because we have an implementation detail inside the trait and we don't want to expose this to the public API. If this trait were allowed to be implemented by foreign types then they would have to be aware that ndarray is in use, etc. and changing this later would be a breaking change.

For the implementation, all of our rank types (0 through 5) follow the same structure, so here is the definition of rank 0 as an example:

```rust
#[derive(Clone, Debug, Default, Eq, PartialEq)]
pub struct Zero;
impl Rank for Zero {
    type Internal = Ix0;
}
impl Sealed for Zero {}
```