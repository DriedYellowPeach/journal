# ch3 Basic Types

## Introduction
the design goal of rust basic types is: safety, efficiency, and concision.

safety:
* use reference rather than raw pointers, reference is never null, never dangling
* no unions, use enum instead

efficiency:
* the bit trick, `Option<&T>` takes up only one Machine Word
* prefer compile-time cost than runtime, prefer generics over abstract object(trait object)

concision:
* the length of integer type is clear, `i32`, `u8`, better than `long long`
* array, vector, slice are different types

## Type Inference
in most situation, you don't need to spell out the type of variables:
```rust
let _a = 5;
let _b: i8 = 127;
```

but in some situation, type must be clearly spell out:
* function parameter
* return type
* struct field type
* when compiler can't infer the type

```rust
let v = (1..5).into_iter().collect::<Vec<i32>>();
```

## Primitive Types
type list:
* numeric
* tuple
* struct, unit struct, tuple-like struct
* enum
* box
* reference(shared and mutable)
* String
* str &str
* array
* slice
* vector
* trait object 
* closure
* function

## Pointer Types
there are basically 3 pointer types:
* reference, shared and mutable
* all kinds of boxes, `box, rc, arc, cell`
* raw pointers, `* const T` `* mut T`

## Sequence Types
there are basically 3 sequence types:
* array, `[T;N]`, element type and lenght are both parts of the type
* vector, `Vec<T>`, heap allocated buffer, dynamical growing
* slice, `&[T], &mut [T]`, fat pointer, referece to sequence

array is always initialized, there is no notation for uninitalized array.
 
## Vector
resizable, element on the heap.
basic operation: `insert, pop, swap_remove, with_capacity`
**hello** 

# Slice
slice is an unsized object, you can only use reference to slice, reference to slice is fat pointer: address + size

can make slice from array or vector, `&v[..]`.

function always consider using slice as arguments, this make function easy to use, slice can point to stack, heap. Many method belong to slice.


# ch4 Ownership

This chapter talks about lifetime of variables, it's really difficult to make it right. The core problem is simple: How to ensure when a variable gone, its memory been freed?

We will see:
* two important promise
* rust's unique feature to make sure variable gone properly:
    * ownership
    * move
    * copy types
    * `Rc` and `Arc`, the shared ownership

## Lifetime

### Two Promise
there are 2 import promise: 
* you decide the lifetime of your variable, the lifetime of any variable is all under control
* your program will never use a pointer to an object after it has been freed. this is called dangling pointer.

c/c++ keep the promise, and this is why they are powerful, they are resource-friendly, but they are unsafe, they can't promise the second.

go/java/python, these languages have GC, you don't control the lifetime of variable, so no dangling pointer issue, no memory leak issue, but they may not that blazingly fast.

here is the pros and cons:
* control lifetime: 
    * pros: powerful, resource-friendly
    * cons: dangling pointer issue, memory leak issue, double free issue
* let GC control the lifetime:
    * pros: easy to use, don't need to care about memory much, have less memory issue
    * cons: slow, not that powerful

But Rust came out in a surprising way: it make both of the promises.

### Some Restrictions
How rust did this? 
* By adding some restrictions, enforced by compile-time check

## Ownership In Rust

### What is ownership? 
Owner is the one who decide the lifetime of a value. when to create and when to drop.

Every value has a single owner(multi owner is a special case). When variable goes out of scope, the value dropped.

For complex value, it forms a tree, at the root of the tree, is the variable, when variable gone, the entire tree gone.

### How Rust extends rigid ownership?
If one variable will be the sole owner of the value, it would be impossible to use to program:
* how to pass to function parameter?
* some value have multi owner?

rust extends the ownership in several ways:
* can move values from one ownership to another, can replace you variable with another value, this is called `Move`, and this allows you to build(move in), rearrange(replace), tear down(move out) value tree.
* the standard library provides the reference-counted pointer types `Arc` and `Rc`, which allows value to have multiple owner.
* you can borrow a reference to a value, reference are non-owing pointer which have limited lifetime.

build and rearrange is easy, tear down need a trick:
* one way is to use `std::mem::replace`.
* another way is to use pattern to destruct `let MyStruct {x, y} = MyStruct{x: vec![], y: vec![]}`, this destruct the struct and move the value into new owner x and y.

these strategies contribute flexibility to the ownership model, while still unholding Rust's promise.

## Moves In Rust

## Copy Types in Rust

## Shared Ownership In Rust
