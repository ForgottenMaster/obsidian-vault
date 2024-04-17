# Overview #

We often find ourselves wanting to find a smaller sequence, inside of a larger sequence. This often manifests as searching for a substring in a larger text, or determining if a list of numbers is a sub-list of another.

We can achieve this by looking at a sequence of characters at the beginning of the larger sequence (known as the haystack) that is the same length as the smaller sequence (known as the needle). This sliding motion can be viewed as the following animation

![Brute Force](Brute%20Force%20String%20Search.gif)

However, with the Naïve approach, when we compare the needle against a specific window in the haystack, we always start by comparing the first elements, then the second elements, and so on until we either hit a mismatch, or have compared all elements as equal.

This is obviously a problem for larger sequences as, every time the window slides one position to the right in the haystack, we can end up checking every character in the needle.

---
# How is Rabin-Karp different? #

The basic movement of the "window" that we're checking in the haystack is still the same - it slides right by one position each step. However where Rabin-Karp differs is by avoiding checking each character to find a mismatch if it knows a mismatch occurs. The way it achieves this is by using a single numeric hash code that represents the sequence of elements we are looking for (known as the fingerprint).

The fingerprint of the initial window can be calculated as usual by hashing all the elements in the window. However, where the efficiency comes from with Rabin-Karp is that the method used to calculate the fingerprint allows us to "roll" that fingerprint each time we move the window to the right by one step.

What this means is, when we move the window to the right by one step, we drop whatever contribution to the overall hash was made by the leftmost element, and then incorporate the new element that's coming in on the right hand side of the window - without having to recalculate all the elements in between.

---
# How can it do this? #

We can do this by multiplying the hash code of each element by a particular, unique number and adding all the values together for a unique fingerprint.

We need to first choose a number as a **base** which we will raise to consecutive powers to determine these multipliers. It doesn't matter too much what this number is, except that it must be greater than 1 (because raising 1 to any power gives 1 which isn't a unique multiplier).

---
# The initial fingerprint #

Given that **B** is the arbitrary numerical base we will raise, **W** is the window we're calculating the fingerprint for, and **L** is the length of that sequence, then the formula is given as follows:

$$ Fingerprint(W) = \sum_{n=0}^{L-1} Hash(W_n) \times B^{L-1-n} $$

Or, written another way:

$$ Fingerprint(W) = (Hash(W_0) \times B^{L-1}) + (Hash(W_1) \times B^{L-1-1}) + (Hash(W_2) \times B^{L-1-2}) $$
$$ + ... + (Hash(W_n) \times B^0 ) $$
---

#### A concrete example ####

It's always difficult to interpret mathematical formulas sometimes, so here is a small and concrete example of calculating the hash code. Given the sequence of integers **$W=[17, 23, 49, 51]$** we can see that **L=4**, and if we pick as our base **B=2** then in order to calculate the fingerprint, we simply do the following:

$$ Fingerprint(W) = (Hash(17) \times 2^3) + (Hash(23) \times 2^2) + (Hash(49) \times 2) + Hash(51) $$

If we furthermore take Hash(X) to be X itself (as they're already integers), then we can easily compute the final fingerprint as:

$$ (17 \times 8) + (23 \times 4) + (49 \times 2) + 51 $$
$$ = 136 + 92 + 98 + 51 $$
$$ = 377 $$
---
# Rolling #

In order to update the fingerprint after the initial calculation, we will need to do the following in sequence:
1. Subtract the highest power term on the left
2. Multiply the fingerprint by B - this has the effect of raising the remaining powers by 1
3. Add the new term to the fingerprint

Let's look at these in turn...

#### Subtract the highest power term ####

When we slide the window to the right, the elements on the left of the window (and the left of the above formula) will "drop off". As we have raised each element to successive powers, we know how to calculate the multiplier added to the hash of element 0. The multiplier is simply $B^{L-1}$, and the value we will subtract from the fingerprint is $Hash(W_0) \times B^{L-1}$

In the concrete example shown above, we subtract $(Hash(17) \times 2^3)$ and are left with:

$$ Fingerprint(W) = (Hash(23) \times 2^2) + (Hash(49) \times 2^1) + (Hash(51) \times 2^0) $$

#### Raise the powers ####

We now multiply by B (in the concrete example this is 2). Since the following is true:

$$ (X + Y) \times Z \equiv (X \times Z) + (Y \times Z) $$

This has the effect of multiplying each term in the formula by B, which furthermore, has the effect of raising the power by 1 of each term.

After multiplying by 2, the concrete example will be given as:

$$ Fingerprint(W) = (Hash(23) \times 2^3) + (Hash(49) \times 2^2) + (Hash(51) \times 2^1) $$

#### Add the new term ####

This is the easiest part. Since the powers of the other terms have been raised, we only need to add the new term into the "zeroeth" power position, which is simply adding the hashcode
of the element.

In our concrete example, say that we are rotating in a new element, 101, on the right as the window slides. The new fingerprint is represented as:

$$ Fingerprint(W) = (Hash(23) \times 2^3) + (Hash(49) \times 2^2) + (Hash(51) \times 2) + Hash(101) $$

Which, again assuming Hash(X) is X, gives the new fingerprint of:

$$ (23 \times 8) + (49 \times 4) + (51 \times 2) + 101 $$
$$ = 184 +  196 + 102 + 101 $$
$$ = 583 $$
---
# But what about overflow? #

As you can see however, from this formula that for large sequence lengths, even for the smallest viable base of 2, will result in an integer growing too large to fit in any sensible datatype.

After all $2^{500000}$ is.....huge

What can we do to keep these numbers small?....***Modular Arithmetic***

#### Modular Arithmetic ####

Using modular arithmetic involves using integer division of the number at each step by some known value (known as the modulus) and retaining the remainder. In this way all the numbers "wrap around" but the formula is still valid, it's just being done on a number circle rather than a number line.

An illustration of this is with clocks. Clocks use a modulus of 12 before they wrap around, but you can still add, for example, 4 hours, or subtract 4 hours when that wraps around.

![Clocks Modular Arithmetic](Modular%20Arithmetic.png)

Adapting the formulas is easy enough, we just apply the modulo operator when calculating the hash code of an element, and at every step of the calculation.

The tricky part of this however, is when we are calculating the powers - we can't calculate the power *and then* apply the modulus as the power calculation overflows. Instead we need to
recognise that:

$$ pow(X, 3) \equiv (X \times X \times X) $$

And this lets us apply modulo at every step, as in (assuming modulus of M):

$$ ((((X \mod M) \times X) \mod M) \times X) \mod M $$

One thing we need to accumulate while doing this is the multiplier we have applied to the left most element ($B^L-1$) which we then can just use later when we're subtracting the left hand term.

---
# Coding it up #

Following is a walkthrough of how to code this algorithm up in Rust, the complete source can be found at the playground link [HERE](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=7fd305a4deb9c822013b7dc77f95975b)

I'll run through the steps one by one below, explaining the code for those who may be unfamiliar with Rust syntax.

#### Step 1 - HashCode trait ####

In Rust we are able to get the hash of a type that implements the *Hash* trait for use in a HashMap. However this method requires passing the type through a *Hasher* and incurring a performance cost to generate a hash code for even basic types such as integers.

Since an integer is its own hash code, and a character can be converted to an integer as a hash code we don't need to do anything too tricky.

For the element hashing therefore, I've gone with creating a new trait:

```rust
trait HashCode {
	fn hash_code(&self) -> u64;
}
```

That is, a simple trait (interface in other languages) with a single function that operates on an const reference to the element (&self) and returns an integer hash code for it (u64).

#### Step 2 - Implementing HashCode for types we want to use ####

The next step is to actually implement the trait for the types we want to be able to use in the sequences for the algorithm. Rust allows us to implement custom traits for existing types in order to extend their functionality - even primitive types. For this, we will implement it for the following primitive types:

1. i32 - A signed 32-bit integer. For this, if it's a positive value we can simply keep the value and cast it to a wider, 64 bit integer with no loss. If it's negative, because we know that the range of a 64-bit integer covers the range of the 32-bit integer then we will map the negative portion of i32 by negating it - and then subtracting from u64::MAX.
2. u32 - This involves a simple widening cast which is safe since we know all unsigned 32-bit integer values will fit into an unsigned 64-bit variable.
3. u64 - For this, the hash code is the value itself so it simply returns it.
4. char - Characters in rust are 4 bytes large representing a Unicode codepoint. For the same reason as u32 will be safely castable into u64, so are characters.

The implementation of the HashCode trait then is as follows:

```rust
impl HashCode for i32 {
    fn hash_code(&self) -> u64 {
        if *self >= 0 {
            *self as u64 // if we're positive, just keep our value and cast as a u64
        } else {
            std::u64::MAX - self.abs() as u64 // if we're negative, just negate it and then subtract from u64 MAX to use the upper end
        }
    }
}

impl HashCode for u32 {
    fn hash_code(&self) -> u64 {
        *self as u64
    }
}

impl HashCode for u64 {
    fn hash_code(&self) -> u64 {
        *self
    }
}

impl HashCode for char {
    fn hash_code(&self) -> u64 {
        (*self).into()
    }
}
```

#### Step 3 - Generating a fingerprint ####

In order to be able to roll a fingerprint as described previously, we first need to construct one from scratch for a given range. Given a sequence of elements (that implement our HashCode trait), along with a base to use, and a modulus to keep the numbers small as described in the section on modular arithmetic - we would like a function that will generate the fingerprint for the sequence. Additionally, we want to return the calculated multiplier for that left-most term so that we don't need to calculate it later.

To allow us to keep the numbers small, we don't use the pow function, but use an imperative loop to accumulate the fingerprint and base offset. 

The code for this function looks like the following (note that this assumes the list isn't empty and will panic if it is due to the [0] access):

```rust
fn generate_fingerprint<T: HashCode>(list: &[T], base: u64, modulus: u64) -> (u64, u64) {
    let mut fingerprint = list[0].hash_code() % modulus;
    let mut base_offset = 1;
    for elem in &list[1..] {
        let elem_hash = elem.hash_code() % modulus;
        fingerprint = (((fingerprint * base) % modulus) + elem_hash) % modulus;
        base_offset = (base_offset * base) % modulus;
    }
    (fingerprint, base_offset)
}
```

#### Step 4 - Rolling the fingerprint ####

In order to "roll" the fingerprint and generate the next one from the previous one, we need the following pieces of information:

1. The previous fingerprint - for obvious reasons
2. The calculated multiplier (called the base offset) for the left-most element - we use this to calculate the final value to subtract from the fingerprint
3. The old element we're rotating out - we need this to get the hash code from, which is combined with the base offset to get the value to subtract
4. The new element we're rotating in - we need this again to get the hash code which will be added to the fingerprint
5. The base - we need this to allow us to "raise the powers" of all the remaining terms in the fingerprint after removing the left most term
6. The modulus - we need this for the same reason as we needed it in **generate_fingerprint**. It lets us perform modular arithmetic and keep the values small

One additional thing to note which could be missed is that before removing the term we want to be rid of, we must first *add* the modulus to the fingerprint. This is because we're using modular arithmetic - it might wrap around so that the current fingerprint value is less than the value we want to subtract. Adding the modulus essentially adds one full rotation to the fingerprint - allowing us to subtract the term safely.

The function looks like the following:

```rust
fn roll_fingerprint<T: HashCode>(
    mut fingerprint: u64,
    base_offset: u64,
    old_term: &T,
    new_term: &T,
    base: u64,
    modulus: u64,
) -> u64 {
    let old_term_hash = old_term.hash_code() % modulus;
    let new_term_hash = new_term.hash_code() % modulus;
    let term_to_subtract = (old_term_hash * base_offset) % modulus;
    fingerprint = fingerprint + modulus - term_to_subtract; // remove the old term after adding the modulus on to protect against underflow.
    fingerprint *= base; // power shift all other terms up by 1
    (fingerprint + new_term_hash) % modulus // return new fingerprint after adding in new term and modding
}
```

#### Step 5 - Putting it together ####

Now we can write the actual Rabin-Karp implementation. This function will take a sequence known as the needle, and an equal or larger sequence known as the Haystack, along with the standard base and modulus to use.

It will return a boolean value indicating whether the needle was found in the haystack or not. This could be extended if necessary in the future to return the index of the match.

For Rabin-Karp, the steps are as follows:

1. Calculate the fingerprint of the needle - this will never change and is the fingerprint we're trying to match
2. Handle the trivial case where the needle is empty - an empty needle is always present in any list
3. Generate the initial fingerprint for the slice of the haystack of the same length as the needle - This is the starting fingerprint to match, and also gives us the base offset to use
4. If they match already, then we return a match
5. Otherwise we slide the window along by 1 each time (until the right side of the window hits the end of the haystack), rolling the hash each time to remove the old left hand element and add the incoming element
6. Each time, check if there's a match
7. If we've checked all windows and not found a match, it's not there

One thing to note though is that it *is* sufficient to detect a mismatch by mismatching fingerprints however it's *not* sufficient to detect a match with matching fingerprints. This is  because we're using modular arithmetic and a mathematical formula for calculating the final fingerprint - it's possible different sequences end up with the same fingerprint.

The efficiency of this algorithm is in the fact we don't have to check elementwise when we know there's a mismatch, and only have to check elementwise in the cases that the fingerprints match.

Because of this requirement that elements need to be compared as equal, our Rabin-Karp function requires an additional bound on it's generic type, that of PartialEq (which lets us use the == operator). The final function is as follows:

```rust
fn rabin_karp<T: HashCode + PartialEq>(
    needle: &[T],
    haystack: &[T],
    base: u64,
    modulus: u64,
) -> bool {
    if needle.len() == 0 {
        true // we can always find the empty list inside any list
    } else {
        // get the initial fingerprints and window
        let (needle_fingerprint, _) = generate_fingerprint(needle, base, modulus);
        let needle_len = needle.len();
        let haystack_len = haystack.len();
        let mut window = &haystack[0..needle_len];
        let (mut window_fingerprint, base_offset) = generate_fingerprint(window, base, modulus);

        // check the initial fingerprints/window for match
        if needle_fingerprint == window_fingerprint && needle == window {
            return true;
        }

        // otherwise run a starting index for the window from 1 up to and including haystack_len-needle_len.
        for window_index in 1..=(haystack_len - needle_len) {
            let new_window = &haystack[window_index..window_index + needle_len];
            let roll_out = &window[0];
            let roll_in = &new_window[needle_len - 1];
            window_fingerprint = roll_fingerprint(
                window_fingerprint,
                base_offset,
                roll_out,
                roll_in,
                base,
                modulus,
            );
            window = new_window;
            if needle_fingerprint == window_fingerprint && needle == window {
                return true;
            }
        }

        // wasn't found or we'd have returned true on a match.
        false
    }
}
```

---
# Testing the implementation #

The following code snippet is used to test the implementation of Rabin-Karp finds a match or not in the correct cases:

```rust
fn main() {
    const BASE: u64 = 253;
    const MODULUS: u64 = 101;
    println!(
        "[2, 4, 1] in [7, 8, 2, 4, 1, 5] => {}",
        rabin_karp(&[2, 4, 1], &[7, 8, 2, 4, 1, 5], BASE, MODULUS)
    );
    println!(
        "[2, 4, 1] in [7, 8, 2, 4, 3, 5] => {}",
        rabin_karp(&[2, 4, 1], &[7, 8, 2, 4, 3, 5], BASE, MODULUS)
    );
}
```

We're using relatively small sequences, and small modulus but this algorithm is pretty efficient when scaled up at large sizes. In my experiments, a haystack of
length 1,000,000 and a needle of length 500,000 resulted in a benchmark time on the naïve algorithm of **over 1 minute**, whereas with the Rabin-Karp implementation
benchmarked the same problem at around **13 milliseconds**

The output from this is:

```
[2, 4, 1] in [7, 8, 2, 4, 1, 5] => true
[2, 4, 1] in [7, 8, 2, 4, 3, 5] => false
```
