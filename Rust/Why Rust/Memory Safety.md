Here we'll go over the memory model in Rust, along with the language design choices that allow for Rust to guarantee memory safety in any programs written using it. We will also be comparing Rust against C++ as they are both low level programming languages that are suitable for systems programming.

---
# Memory guarantees #

The Rust language was designed with memory safety in mind and as such, the following guarantees are **always** true in a Rust program:
1. A value has only one owner in the code at a given time.
2. When the owner of a value goes out of scope, the value is dropped.
3. A value cannot be referenced after it's dropped.
4. A value cannot have ownership transferred while there are still references taken to it.
5. A value can only be mutably referenced if there are no other references.

In contrast, C++ guarantees the following:
* No memory safety guarantees

The design decisions taken by the Rust language team in developing the language mean that the following types of bugs don't exist in Rust:
1. Use after free
2. Double free
3. Modification while reading (e.g. changing collection while iterating)
4. Data races (multiple threads accessing same memory location at the same time)

---
# Disclaimer #

Rust has a method to allow the developer to write unsafe code, and ensures that developers know exactly which portions of the code have been manually checked for correctness versus those portions of the code that have been determined to be correct by the compiler.

When writing unsafe Rust, it is the developers responsibility to ensure that everything they do in that unsafe block upholds Rust's memory safety guarantees. Safe Rust must be able to assume that unsafe Rust code blocks are correct.

The above memory safety guarantees therefore are only valid if all the unsafe code blocks are correct and also following the guarantees. It's 100% possible to introduce the above behaviours and therefore undefined behaviour by abusing the powers granted by the unsafe keyword.

---
# The problems in C++ and how to solve them #

As mentioned in the Memory guarantees section, Rust guarantees that invalid memory is never used, and also guarantees data races where one source can be writing to a memory location while another is reading from it (multiple readers is fine).

This section will go over the 4 big issues which the C++ compiler doesn't do anything to help with, and how the Rust compiler will catch them.

#### 1. Use after free ####

C++
```cpp
#include <iostream>
using namespace std;

int main() {
	string* s = new string{"Hello, World!"};
	delete s;
	s->push_back('c');
	return 0;
}
```

Result (runtime)
```
Runtime error #stdin #stdout #stderr 0.01s 5436KB
```

Rust
```rust
fn main() {
    let mut s = String::from("Hello, World!");
    drop(s);
    s.push('c');
}
```

Result (compile time)
```
error[E0382]: borrow of moved value: `s`
 --> src/main.rs:4:5
  |
2 |     let mut s = String::from("Hello, World!");
  |         ----- move occurs because `s` has type `String`, which does not implement the `Copy` trait
3 |     drop(s);
  |          - value moved here
4 |     s.push('c');
  |     ^^^^^^^^^^^ value borrowed here after move
```

#### 2. Double free ####

C++
```cpp
#include <iostream>
using namespace std;

int main() {
	string* s = new string{"Hello, World!"};
	delete s;
	delete s;
	return 0;
}
```

Result (runtime)
```
Runtime error #stdin #stdout #stderr 0.01s 5432KB
```

Rust
```rust
fn main() {
    let s = String::from("Hello, World!");
    drop(s);
    drop(s);
}
```

Result (compile time)
```
error[E0382]: use of moved value: `s`
 --> src/main.rs:4:10
  |
2 |     let s = String::from("Hello, World!");
  |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
3 |     drop(s);
  |          - value moved here
4 |     drop(s);
  |          ^ value used here after move
```

#### 3. Modification while reading ####

C++
```cpp
#include <iostream>
#include <vector>
using namespace std;

int main() {
	vector<int> v;
	v.push_back(1);
	v.push_back(2);
	v.push_back(3);
	for (int& i : v)
	{
		v.reserve(10); // causes reallocation, while iterating over the vector
		cout << i << endl;
	}
	return 0;
}
```

Result (runtime)
```
Success #stdin #stdout 0.01s 5516KB
-1913700720
21879
-1913774064
```

Rust
```rust
fn main() {
    let mut v = Vec::with_capacity(3);
    v.push(1);
    v.push(2);
    v.push(3);
    for i in &v {
        v.reserve(10);
        println!("{i}");
    }
}
```

Result (compile time)
```
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 --> src/main.rs:7:9
  |
6 |     for &i in &v {
  |               --
  |               |
  |               immutable borrow occurs here
  |               immutable borrow later used here
7 |         v.reserve(10);
  |         ^^^^^^^^^^^^^ mutable borrow occurs here
```

#### 4. Data races ####

Rust prevents data races by using two marker interfaces which are automatically implemented for types that are safe to send to a different thread or to share between threads. When writing custom types that should be threadsafe using **unsafe** code, you may need to manually implement these to tell the compiler that it can assume that these are thread safe and that implementation was checked by the developer.

From the point of view of safe Rust though, it can use these traits/interfaces to identify that all types being used in a multithreaded situation are in fact marked as being thread safe.

These traits are called Send and Sync.

**Send** identifies a type as being safe to have its ownership transferred to another thread.
**Sync** identifies a type as being safe to have *references* transferred to another thread.

A type that is marked as Sync is essentially saying that it's proven to be safe and correct to access the methods from multiple threads at the same time.

This kind of markup is something that C++ is missing. C++ allows any types to be used and accessed from multiple threads which results in potential data races as multiple threads write to the same memory location at the same time.

By allowing a method of flagging a type as being thread safe, and only allowing those types to be used in a multithreaded environment, the possibility of a data race is eliminated.

The example below does not compile because we're trying to share an Rc (reference counted heap based ownership) between threads which is not flagged as thread safe because the reference count being used is not atomic:

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let v = Rc::new(42);
    thread::scope(|s| {
        s.spawn(|| println!("{}", Rc::clone(&v)));
        s.spawn(|| println!("{}", Rc::clone(&v)));
    });
}
```

Results in the compiler error:
```
error[E0277]: `Rc<i32>` cannot be shared between threads safely
 --> src/main.rs:7:17
  |
7 |         s.spawn(|| println!("{}", Rc::clone(&v)));
  |           ----- ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ `Rc<i32>` cannot be shared between threads safely
  |           |
  |           required by a bound introduced by this call
  |
  = help: the trait `Sync` is not implemented for `Rc<i32>`
  = note: required for `&Rc<i32>` to implement `Send`
```

However, using an **Arc** (atomic reference counted heap allocated value) is thread safe, specifically because the reference count being used is atomic and thus safe for sending to multiple threads:

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let v = Arc::new(42);
    thread::scope(|s| {
        s.spawn(|| println!("{}", Arc::clone(&v)));
        s.spawn(|| println!("{}", Arc::clone(&v)));
    });
}
```

We then get the correct output

```
42
42
```