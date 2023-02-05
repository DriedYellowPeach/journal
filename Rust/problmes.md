
## why Option<T> is Copy?

in source code
```rust
#[derive(Copy, ...)]
enum Option<T> {

}
```

but in the documents, it wrote
```rust
impl<T> Copy for Option<T> 
where T:Copy
{

}
```

so how derive works? 
Option<T> is indeed Copy in some situation.

in docs for Copy, it is said: There is a small difference between the two: the derive strategy will also place a Copy bound on type parameters, which isn’t always desired. 

and it looks like PhantomData<T> is this situation, even if T is not Copy, PhantomData is Copy

## what is PhantomData? 
seems like to enforce compile time rules but looks like runtime things. 
often used to enforce a lifetime parameters on struct, and the usage of this struct must under control!
