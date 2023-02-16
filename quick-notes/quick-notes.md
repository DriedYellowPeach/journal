
- [date] Mon Feb 6 2023
    - https://gitlab.torproject.org/tpo/core/arti/-/blob/main/doc/dev/Architecture.md, why use dyn trait? why use weak?

- [date] Tue Feb 7 2023
    - what does pub(self) pub(crate) pub(super)

- [date] Wed Feb 8 2023
    - what is Monad?
        - wrap type, wrap function, bind function.
        - Some abstract example of Using monad: Option, writer, List is a kind of monad, future/promise monad
    - Box<dyn Error> from Error: how this be done? the ? operator did this convertion automatically.
    - usage of crate: anyhow, easy to use error handling
    - usage of crate: clap, command line application
    - the impl of Tree data structure in rust, why use Box, and Why Box is the same as pointer in c, c++, go
    - leetcode 1233, learn rust cargo benchmark.

- [date] Thu Feb 9 2023
    - solve clangd not working: can't find some headers, this is because lsp need to read cmake to get things to work, may solution is : add -DCMAKE_EXPORT_COMPILE_COMMANDS=1, then, it will generate a compile_commands.json file , which could be automatic read by clangd. and i see no more warnings for headers, ln -s build/compile_commands.json . and this would be useful
    - try use rust-lldb to debug multi-thread program.
    - what does `Box<dyn Error + 'a>`  means? Does Box have lifetime? Box of reference have lifetime?
        - this is called lifetime as trait bounds. Box<E> E: ERROR + 'a, this mean value of type E should have lifetime > 'a, but also means 'a lifetime in the return value should less than lifetime of value of type E.
        - after the lines of call this 'from' function, the trait object contains the lifetime, deref the box, get a reference, 
        - box &**
        - T is of type &'a I, deref result is &'b &'a I. this is how Box<dyn E + 'a> get into effect.
        - write a test about this. still can use box, but can't use inside box?
        - Box<T> have lifetime of T, so Box<'a i32> have lifetime 'a
        - use raw pointer, always bingding a lifetime with, just like reference, but handmade, MyRawPointer<'a, T:'a>.
        - generics Box<T> and what if T is pointer type(a reference), Box<'a>(Unique::(&'a i32)); double generic, generic lifetime of generic type
    - the turbofish syntax in rust struct expression `MyStruct::<T> {}`

- [date] Thu Feb 9 2023
    - what is the difference between `S<'a, T:'a>` and `S<'a>` and `S<'a, 'b, T:'b>` 

- [weekend todo]
    - lifetime parameter as bound, from Error to Box<dyn E + 'a>
    - rust-lldb multi-thread debug
    - dictionary tree
    - 

- [date] 02/14/2023
    - rust object safety, trait that use Self or required sized object are not designed for trait object, and can't use them.

- [date] 02/15/2023
    - custom snippet using friendly snippet. learn how to use this plugin
    - rust object safety, trait that use Self or required sized object are not designed for trait object, and can't use them. The idea of "object-safety" is that you must be able to call methods of the trait the same way for any instance of the trait. So the properties guarantee that, no matter what, the size and shape of the arguments and return value only depend on the bare trait — not on the instance (on Self) or any type arguments (which will have been "forgotten" by runtime).
