# A way to self-referential structs in Rust

Self-referential structs have long been a challenge in Rust. This document discusses some of my ideas on how that would be possible, showing why it has to be so complex, but while thinking about the problem I got some interesting ideas like out-of-lifetime references which might be of practical interest.

## First step: allowing references to exist outside their lifetime

The first problem in creating a self-referential struct is the "chicken or the egg" problem. Assuming we have a struct { `field1`, `field2` } where `field2` points to `field1`. In Rust all fields have to be initialized before the struct can be used, but if field2 points to field1 then the field1 must be available before we can write field2. The "chicken" here is the struct, while the "egg" is the `field2`. Which one to initialize first?

It turns out that the paradox can be solved by allowing out-of-lifetime references.

Let's look at this piece of code:

```rust
let list_owned: Vec<String>; // assume we get this variable somewhere
let list_ref: Vec<(&str, Metadata)> = list_owned.iter().map(|s| (s.as_str(), get_metadata(s))).collect();
do_something_with(list_ref);
drop(list_owned);
for let (s, metadata) in list_ref { // error: list_owned borrowed while still dropped
    do_something_with(metadata);
}
```

We allocate a vector `list_owned` of owned strings, create `list_ref` from `list_owned`and drop `list_owned`. The problem with Rust is that, because we have a string reference in `list_ref`, `list_ref` is bound to a lifetime shorter than `list_owned`. But these string references are not all that `list_ref`  contains! It also contains an allocation to an array of tuples, and some metadata info, which are still useful even after `list_owned`'s lifetime.

Currently, a struct's lifetime is always the intersection of all its fields, and every field effectively has `'static` lifetime. This ensures that if you have a reference to the struct, you have a reference to all its fields. However, this prevents safe code like above. We extend the concept of references and allow references to *exist* outside their lifetime, even though they will not be usable, and call such references *out-of-lifetime* references. By allowing out-of-lifetime references, this is no longer the case; we can have a struct with a longer lifetime than some of its fields, though the field is just not accessible outside its lifetime.

Why can we do such an extension? In fact, the existence of a reference does not imply that the reference is usable; Rust currently enforces that references must be either dropped or forgotten once they run out of lifetime, but it is not in fact necessary for code to be safe. A slightly modified example from https://doc.rust-lang.org/nomicon/lifetimes.html showing the concept:

```rust
let x = 0;
let z = &x;
let y = &x;
z = y;

'a: {
    let x: i32 = 0;
    'b: {
        let z: &'c i32 = &'c x; // As long as we don't use this reference outside 'c, why can't it exist here?
        'c: {
            // We don't have to use 'b here!
            let y: &'c i32 = &'c x;
            z = y;
        }
    }
}
```

This allows us to write code like this:

```rust
let list_owned: Vec<String>; // assume we get this variable somewhere
// 'a starts
let list_ref: Vec<(&'a str, Metadata)> = list_owned.iter().map(|s| (s.as_str(), get_metadata(s)).collect();
// For some reasons we don't want to drop list_ref right away. Maybe the Metadata contains a drop handler that takes too long to run, or we want just the metadata and not the strings. I don't know if a specific use case exists, but the current behavior is preventing us from writing safe code. As long as we don't dereference the string outside 'a, we should be actually safe!.
// 'a ends
drop(list_owned);
// At this point, we are outside 'a. We can still use list_ref safely as long as we do not dereference a borrow with lifetime 'a outside 'a.
for let (s, metadata) in list_ref {
    do_something_with(metadata); // safe!
    println!("{}", s); // error: dereferencing an out-of-lifetime reference
}
```

But there is a new problem: nearly all functions that take references would break because they do not even know whether the reference is usable (i.e. in their lifetime). For example:

```rust
let list_owned: Vec<String>; // assume we get this variable somewhere
// 'a starts
let list_ref: Vec<(&'a str, Metadata)> = list_owned.iter().map(|s| (s.as_str(), get_metadata(s)).collect();
// For some reasons we don't want to drop list_ref right away. Maybe the Metadata contains a drop handler that takes too long to run, or we want just the metadata and not the strings. I don't know if a specific use case exists, but the current behavior is preventing us from writing safe code. As long as we don't dereference the string outside 'a, we should be actually safe!.
// 'a ends
drop(list_owned);
// At this point, we are outside 'a. We can still use list_ref safely as long as we do not dereference a borrow with lifetime 'a outside 'a.
for let (s, metadata) in list_ref {
    do_something_with(metadata); // safe!
    work(s); // What happens if we try to pass an unusable reference to a function?
}
fn work<'a>'(s: &'a str) {
    // 'a could be outside its lifetime, so we could not dereference self at all!
}
```

This is because most functions expect that reference arguments are usable while the function is running, but the fact that the reference argument's lifetime includes that of the function is no longer guaranteed when out-of-lifetime references come into the game. We introduce two new lifetime identifiers, called `'none` and `'self`. `'none` means that the reference is out-of-lifetime, and `'self` means the lifetime shall remain usable during the execution of the function. This allows us to distinguish between a reference that is *usable* and an out-of-lifetime reference. The definition of `'self` can be naturally extended to structs to mean that the reference is usable as long as the struct exists and is not moved. In this way, the above function can be written as:

```rust
fn work<'a: 'self>(s: &'a mut str) {
    'self: {
        // s is usable within the function body
    }
}
```

All lifetime parameters should automatically take `'self`  unless explicitly specified as "could be `'none`". This ensures that currently working functions continue to behave as original.

Structs that contain references also take an implicit lifetime of all lifetime parameters defined on the struct. For example:

```rust
struct VecIterator<'a, T> {
    v: &'a Vec<T>,
    index: usize,
}
impl<'a, T> VecIterator<'a, T> {
    // This needs to take 'a in addition to 'self.
    fn next<'b: 'self  + 'a>(&'b mut self) -> Option<&'a T>
    {
        // ...
    }
    // But the struct can still be used even after 'a! 'a is only necessary for dereferencing "v" but not necessary for accessing any field (it's OK to access "v" as long as it's not dereferenced). Such usage will need to be explicitly declared though.
    fn get_index(&'self self) -> usize {
        self.index
    }
}
```

Note that we are using the operator `+` for lifetimes. The lifetime `'d+'a` means that the reference can only be used when the code is executing in both `'d` and `'a`; i.e. the real lifetime is the *intersection* of `'d` and `'a`, not the union of  `'d` and `'a`.  Since we allow references to exist outside their real lifetime, the load address operation can take place outside 'a (but not outside 'd); it's just that it can only be *used* within `'d+'a`. 

## Expressing borrow relationship

With this change, we're one step closer to a self-referencing struct, though it doesn't quite work yet. Consider the following code:

```rust
struct SelfReferential {
    buf: Vec<u8>,
    cur: Option<&'self mut Vec<u8>>,
}
// Since we allow out-of-lifetime references now, we can keep data's lifetime 'static rather than 'self.
// This allows us to construct the struct outside the 'self lifetime, when the reference contained in `cur` field cannot exist yet.
let data = SelfReferential { buf: vec![0, 1, 2, 3], cur: None };
// Now we entered the 'self lifetime. Let's try to borrow a field!
// 'data start
data.cur = Some(&'data mut data.buf); // works
send_data(data); // Doesn't work! The borrow checker is only able to track borrow status within the current stack frame, but such information cannot be tracked across the function call boundary. Instead, it only knows how to transfer ownership of the whole struct. However, since data.buf is borrowed by data.cur, and we cannot move borrowed fields, we cannot move data around yet. If we forcibly try to move it, the send_data function will forget the fact that buf is borrowed by cur, and violate memory safety.
// This means that a self-referencing struct can only be used within a single stack frame, which is practically useless.
// 'data end
```

In order for this to work, we need to have the borrow checker check the struct, just like how it checks a stack frame.

The problem also arises in other contexts. For example, how to implement the following code without `unsafe`?

```rust
use tokio::sync::{Mutex, MutexGuard};
use tokio::time::{delay_for, Duration};
use std::sync::Arc;

#[tokio::main]
async fn main() {
	let obj = Arc::new(Mutex::new(1usize));
	let obj2 = Arc::clone(&obj);
    let lock: MutexGuard<'_, usize> = obj2.lock().await;
    let mut lock = unsafe { std::mem::transmute::<MutexGuard<'_, usize>, MutexGuard<'static, usize>>(lock) };
    tokio::spawn(async move {
        // We want to move the MutexGuard into the block.
        // MutexGuard has a reference to the object behind Arc, so we must move Arc here too to ensure that the object exists.
        delay_for(Duration::new(1, 0)).await;
        *lock += 1;
        drop(obj2);
    }).await.unwrap();
    let lock = obj.lock().await;
    assert!(*lock == 2);
}
```

`lock` borrows `obj2` so it cannot be moved on its own. We expect to be able to move both `lock` and `obj2`  together, but it's not currently possible. In order to make the code safe, we need to specify at the closure boundary that `lock` borrows `&Mutex<usize>` and the existence of `obj2` guarantees that the `Mutex<usize>` will stay in its place.

Let's observe what's happening on a slightly complex stack frame. In order to understand what's going on, we split the borrow into two parts: take a reference of `Box<usize>` first, and then use that reference and ask `Box::borrow_mut` to "exchange" that reference for a mutable reference of what's inside the `Box`. We introduce a hypothetical `frame: &mut Frame` reference, which corresponds to the stack pointer, like `RBP` on x86-64 architectures for example, and have all local variables stored inside the `Frame` struct.

```rust
fn test(frame: &'self mut Frame) {
    // 'self begin
    // 'data1 begin
    let frame.data1 = Box::new(5usize); // Label 0
    // 'data2 begin
    let frame.data2 = Box::new(8usize);
    // 'data1 and 'data2 end; 'ptr1 begin
    let frame.data1_ptr = &mut frame.data1; // Label 1.
    let frame.ptr1 = frame.data1_ptr.borrow_mut(); // Label 2
    // 'ptr1 end; ptr2 begin
    let frame.ptr2 = &mut frame.ptr1;
    let frame.data2_ptr = &mut frame.data2;
    *frame.ptr2 = frame.data2_ptr.borrow_mut(); // Label 3
    // 'ptr2 end
    // 'data1 end, 'data2 end
    // 'self end
}
```

This corresponds to the following `Frame` struct:

```rust
struct Frame {
    data1: Box<usize>,
    data2: Box<usize>,
    data1_ptr: &'data1_ptr mut Box<usize>,
    ptr1: &'ptr1 mut usize,
    ptr2: &'ptr2 mut &'ptr1 mut usize,
    data2_ptr: &'data2_ptr mut Box<usize>,
}
```

Let's ignore the fact that all fields have to be initialized for a moment, and assume that the compiler is clever enough to understand which fields are initialized and which fields are not. We'll discuss the constraints on the lifetimes `'data1_ptr`, `'ptr1`, `'ptr2` , `'data2_ptr` and `'self` first. At the beginning of `fn test()`, none of the fields are initialized.

At Label 0, we dereference `&'self mut Frame`, so we check that we are really within 'self. Then we use the reference to assign a variable with lifetime 'static. However, since the reference is only used to modify a value and not stored, no variable inherits lifetime `'self`.

At Label 1, we dereference `&'self mut Frame`, and then creates a reference `data1_ptr` from it. Such a reference must be at least lifetime 'self, in addition to its scope, so we get a lifetime constraint: `'data1_ptr: 'self`.

At Label 2, we send `data1_ptr: &'data1_ptr mut Box<usize>` to `Box::borrow_mut()` and get `ptr1: &'data1_ptr mut usize` back. Since data1_ptr's lifetime is `'data1_ptr`, we need to check that we are in `data1_ptr`, and we indeed are. Since we need to assign `&'data1_ptr mut usize` to `&'ptr1 mut usize`, we need a constraint: `'data1_ptr: 'ptr1`.

At Label 3, the type `&'ptr1 mut usize` is assigned with a value of type `&'data2_ptr mut usize`, thus we find another constraint:

* `'data2_ptr`: `'ptr1`

Thus the full list of lifetime constraints include:

* `'data1_ptr`: `'self`
* `'data2_ptr`: `'self`
* `'data1_ptr`: `'ptr1`
* `'data2_ptr`: `'ptr1`
* `'ptr2`: `'self`

The relationship between lifetimes can be expressed using a directed graph, and forms a transitive closure. 

With lifetimes analyzed, we can start thinking about how borrows can be checked. According to https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html#the-rules-of-references, the rules of references are:

- At any given time, you can have *either* one mutable reference *or* any number of immutable references.
- References must always be valid.

The rules are a bit too abstract because it omits referents. We need to find a way to identify all the referents. We introduce a new symbol `#` at the prefix of a type or a variable, which means the identity of the variable within the current context.

```rust
struct Frame {
    data1: Box<usize>,
    data2: Box<usize>,
    data1_ptr: &'data1_ptr mut Box<usize>,
    ptr1: &'ptr1 mut usize,
    ptr2: &'ptr2 mut &'ptr1 mut usize,
    data2_ptr: &'data2_ptr mut Box<usize>,
}
let f1: Frame;
let f2: Frame;
// #f1: #Frame
#f1 == #f2 // false
(#f1).data1 == (#f2).data1 // false
let f3: &mut Frame;
#f3 == f3 // true
#f3.data1 // INVALID! # can only be used to identify a direct struct field, not through a reference.
let f4: Box<Frame>;
#f4.data1 // Invalid as well because Box<Frame> has to be converted to &mut Frame before the struct fields can be used.
```

Then we can apply the rules by iterating through every reference and check the objects that it could potentially borrow. For every `f: #Frame<'data1_ptr, 'ptr1, 'ptr2, 'data2_ptr>`,

* `f.data1_ptr` borrows `f.data1` mutably during  `'data1_ptr`
* Contrary to intuition, `f.ptr1` does not borrow anything; the only reason it can contain a reference is because `Box::borrow_mut` gave us one
* `f.ptr2` borrows `f.ptr1` mutably during `'ptr2`
* `f.data2_ptr` borrows `f.data2` mutably during `'data2_ptr`

We sort those constraints according to referent instead of referer:

* `f.data1` is mutably borrowed by `f.data1_ptr` during `'data1_ptr` only
* `f.data2` is mutably borrowed by `f.data2_ptr` during `'data2_ptr`only
* `f.ptr1` is mutably borrowed by `f.ptr2` during `'ptr2` only

The borrow checker essentially checks that for every referent:

* The lifetimes during which the object is mutably borrowed do not overlap with each other
* The lifetimes during which the object is immutably borrowed do not overlap with that of which the object is mutably borrowed

Thus, our annotation just needs to contain the above information, written this way:

```rust
struct Frame<'data1_ptr, 'ptr1, 'ptr2, 'data2_ptr>
where
	'data1_ptr: 'self + 'ptr1,
	'data2_ptr: 'self + 'ptr1,
	'ptr2: 'self,
	#data1: mutably-borrowed('data1_ptr),
	#data2: mutably-borrowed('data2_ptr),
	#ptr2: mutably-borrowed('ptr2),
{
    data1: Box<usize>,
    data2: Box<usize>,
    data1_ptr: &'data1_ptr mut Box<usize>,
    ptr1: &'ptr1 mut usize,
    ptr2: &'ptr2 mut &'ptr1 mut usize,
    data2_ptr: &'data2_ptr mut Box<usize>,
}
```

Note that we do not care about *when* the borrow happened; for example,  the line`#data1: mutably-borrowed('data1_ptr)` means that at some point in time we borrowed the field `data1` mutably, but it does not need to happen at the beginning `'data1_ptr`; it can happen in the middle or even before `'data1_ptr`, like the piece of code at the top of the document. We do care about the number of times it happened; a mutable borrow can only happen once for one lifetime, for example, because otherwise the lifetime would overlap with itself, violating the first rule (except when the lifetime is `'none`).

At every point of execution, every lifetime parameter can be either `'none` or `'self` . In order to describe non-existent fields statically, we introduce another kind generic parameters with boolean values, prefixed with `?`. It denotes whether a field exists at all:

```rust
struct Frame<'data1_ptr, 'ptr1, 'ptr2, 'data2_ptr, ?data1, ?data2, ?data1_ptr, ?ptr1, ?ptr2, ?data2_ptr>
where
	'data1_ptr: 'self + 'ptr1,
	'data2_ptr: 'self + 'ptr1,
	'ptr2: 'self,
	#data1: mutably-borrowed('data1_ptr),
	#data2: mutably-borrowed('data2_ptr),
	#ptr2: mutably-borrowed('ptr2),
{
    data1: MaybeExist<?data1, Box<usize>>,
    data2: MaybeExist<?data2, Box<usize>>,
    data1_ptr: MaybeExist<?data1_ptr, &'data1_ptr mut Box<usize>>,
    ptr1: MaybeExist<?ptr1, &'ptr1 mut usize>,
    ptr2: MaybeExist<?ptr2, &'ptr2 mut &'ptr1 mut usize>,
    data2_ptr: MaybeExist<?data2_ptr, &'data2_ptr mut Box<usize>>,
}
```

```rust
fn test(frame: &'self mut Frame) {
    // Frame<'none, 'none, 'none, 'none, false, false, false, false, false, false>
    // The compiler needs to turn ?data1 to true after the statement.
    let frame.data1 = Box::new(5usize);
    // Frame<'none, 'none, 'none, 'none, true, false, false, false, false, false>
    // Similarly, the compiler needs to turn ?data2 to true after the statement.
    let frame.data2 = Box::new(8usize);
    // Frame<'none, 'none, 'none, 'none, true, true, false, false, false, false>
    // The borrow can happens within the struct context with the rule `#data1: mutably-borrowed('data1_ptr)`.
    // The compiler needs to somehow exploit the rule and begin 'data1_ptr here to have the borrow happen
    // on #frame rather than #self.
    let frame.data1_ptr = &mut frame.data1;
    // Frame<'self, 'none, 'self, 'none, true, true, true, false, false, false>
    let frame.ptr1 = frame.data1_ptr.borrow_mut();
    // Frame<'self, 'self, 'none, 'self, true, true, false, true, false, false>
    // Note that even though data1_ptr no longer exists, it still has to be 'self because it can be assigned into 'ptr1 which is 'self. This is how ptr1's unique mutabilitity to data1 is enforced - it is indirectly borrowed through data1_ptr.
    // data2_ptr as well - even though it is not used until the last line, we have the constraint 'data2_ptr: 'self + 'ptr1 which causes the same thing.
    let frame.ptr2 = &mut frame.ptr1;
    // Frame<'self, 'self, 'self, 'self, true, true, false, true, true, false>
    let frame.data2_ptr = &mut frame.data2;
    // Frame<'self, 'self, 'self, 'self, true, true, false, true, true, true>
    *frame.ptr2 = frame.data2_ptr.borrow_mut();
    // Frame<'self, 'self, 'self, 'self, true, true, false, true, true, false>
}
```

Borrow check is equivalent to checking the two rules for every referent for every `Frame` type signature occurring in the function body.

## Hierarchical move

Back to this piece of code:

```rust
use tokio::sync::{Mutex, MutexGuard};
use tokio::time::{delay_for, Duration};
use std::sync::Arc;

#[tokio::main]
async fn main() {
	let obj = Arc::new(Mutex::new(1usize));
	let obj2 = Arc::clone(&obj);
    let lock: MutexGuard<'_, usize> = obj2.lock().await;
    let mut lock = unsafe { std::mem::transmute::<MutexGuard<'_, usize>, MutexGuard<'static, usize>>(lock) };
    tokio::spawn(async move {
        // We want to move the MutexGuard into the block.
        // MutexGuard has a reference to the object behind Arc, so we must move Arc here too to ensure that the object exists.
        delay_for(Duration::new(1, 0)).await;
        *lock += 1;
        drop(obj2);
    }).await.unwrap();
    let lock = obj.lock().await;
    assert!(*lock == 2);
}
```

We want to express the fact that we can move `lock` as long as we move `obj` or `obj2` together. The problem here is that `lock` contains a reference to `&'a Mutex<usize>`, which is coerced from `&'a Arc<Mutex<usize>>`, where `'a` is bound to a stack frame on `main`. However,`&Mutex<usize>` is not actually bound to `'a`; it is bound to the Arc allocation on the heap, while `Arc` itself can be freely moved. That is, Arc's coercion signature:

```rust
fn deref(&'a Arc<T>) -> &'a T
```

hides the fact that the lifetime of `&'a Arc` does not have to be contiguous. If you have `Arc` at one point of time, and then move it somewhere else, the referee `T` is still guaranteed to be at the same place.

I haven't thought of a way to express such an noncontiguous lifetime yet, but it should be one of the problems that self-referencing structs are able to solve. One can pack the `Arc` and `MutexGuard` onto one single struct, express the borrow relationship that `MutexGuard` depends on `Arc` , and send the whole struct wrapped in a `Pin`.

## Expressing a self-referential struct

In order to express the above struct, we used a whopping 10 parameters in its definition!

`struct Frame<'data1_ptr, 'ptr1, 'ptr2, 'data2_ptr, ?data1, ?data2, ?data1_ptr, ?ptr1, ?ptr2, ?data2_ptr>`

This is too complex for anyone to write or to understand easily, not to mention it exposes too many details. Instead, a self-referential struct can be created from a regular function. We can add "breakpoints" in the function body at which a new self-referential struct type is created from the current stack frame.

TBD

# Conclusion

Self-referential structs are necessarily complex to understand and to express. However, the first proposal to allow out-of-lifetime references might be a good starting point, since it makes sense in many other contexts, and it would help us find new use cases and new ideas.
