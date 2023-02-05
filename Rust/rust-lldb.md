# Inside Rust Trait Object

- [date] Sun Feb 5 2023
- [about] trait object, lldb, debug, vtable

Looking inside rust trait object. Using the tool - `rust_lldb` to inspect the detail of trait object.

---

Define a simple trait, a struct implement this trait.
```rust
use std::f64::consts::PI;


trait Shape {
    fn description(&self) -> String;
    fn area(&self) -> f64;
    fn perimeter(&self) -> f64;
}

trait ReShape {
    fn bigger(&mut self, size: f64) -> f64;
}

struct Circle {
    radius: f64,
}

impl Circle {
    fn new(radius: f64) -> Self {
        Circle { radius }
    }
}

impl ReShape for Circle {
    fn bigger(&mut self, size: f64) -> f64 {
        self.radius += size;
        self.radius
    }
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        PI * self.radius * self.radius
    }

    fn perimeter(&self) -> f64 {
        PI * 2.0 * self.radius
    }

    fn description(&self) -> String {
        "shape is circle".to_string()
    }
}

fn main() {
    // println!("hello trait object");
    let mut circle = Circle::new(3.0);
    // let shape : Box<dyn Shape> = Box::new(circle);
    let shape : &dyn Shape = &mut circle;
    println!("shape => {}", shape.area());
    let reshape: &mut dyn ReShape = &mut circle;
    println!("reshape => {}", reshape.bigger(1.0));
    println!("end");
}
```

---

## Details of Trait Object
A trait object is actually a fat pointer: `address` and `vtable`. Use `p shape` lldb command to show its field:
```
(lldb) p shape
(&dyn inside_trait_object::Shape) $2 = {
  pointer = 0x000000016fdfedf8
  vtable = 0x0000000100048270
}
(lldb) me read -c32 0x016fdfedf8
0x16fdfedf8: 00 00 00 00 00 00 08 40 00 00 00 00 00 00 00 00  .......@........
0x16fdfee08: 40 ee df 6f 01 00 00 00 a0 82 04 00 01 00 00 00  @..o............
```

We took a look at the `pointer`  field at first:
`pointer` is `0x016fdfedf8`, read forward 8 bytes of the memory from this address, data is `0x40 08 00 00 00 00 00 00`. To prove this bit pattern is float point 3.0: 
```
3 => 0x0011 => 0x11 * 2
E = 1, M = 3/2 
e = E + bias = 1024 => 100 0000 0000
M = 1 + f => f = 1/2 => 1000 ...(52 bit in total)
s = 0 
expoent = [s][e] = 0x40 0
fraction = [f] = 0x8 00 00 00 00 00 00(..)
```

Then we look at the second field `vtable`, we examine the memory the same way:
```
(lldb) me read -c64 0x0100048270
0x100048270: 10 55 00 00 01 00 00 00 08 00 00 00 00 00 00 00  .U..............
0x100048280: 08 00 00 00 00 00 00 00 3c 4a 00 00 01 00 00 00  ........<J......
0x100048290: ec 49 00 00 01 00 00 00 18 4a 00 00 01 00 00 00  .I.......J......
0x1000482a0: ff a5 03 00 01 00 00 00 09 00 00 00 00 00 00 00  ................
```

The memory view is actually this:
```
+------------+-------------+
| destructor |    size     |
+------------+-------------+
| alignment  | description | 
+------------+-------------+
|    area    |  perimeter  | 
+------------+-------------+

+-------------+    +--------------+   
| destructor  |    | 0x0100005510 |   
+-------------+    +--------------+   
| size        |    | 0x0000000008 |   
+-------------+    +--------------+   
| alignment   |    | 0x0000000008 |   
+-------------+    +--------------+   
| description |    | 0x0100004a3c |   
+-------------+    +--------------+   
| area        |    | 0x01000049ec |   
+-------------+    +--------------+   
| perimeter   |    | 0x0100004a18 |   
+-------------+    +--------------+   
```

To prove this, we look at the symbol table and find the address of the function by name:
```
(lldb) image lookup -n "description"
1 match found in /Users/neil/source/rust-playground/lab/target/debug/inside_trait_object:
        Address: inside_trait_object[0x0000000100004a3c] (inside_trait_object.__TEXT.__text + 2132)
(lldb) image lookup -n "area"
1 match found in /Users/neil/source/rust-playground/lab/target/debug/inside_trait_object:
        Address: inside_trait_object[0x00000001000049ec] (inside_trait_object.__TEXT.__text + 2052)
(lldb) image lookup -n "perimeter"
1 match found in /Users/neil/source/rust-playground/lab/target/debug/inside_trait_object:
        Address: inside_trait_object[0x0000000100004a18] (inside_trait_object.__TEXT.__text + 2096)
```

---

## More About Debugging With LLDB

Here are some useful command:
* reload the target: `file [target-path]`
* add breakpoint: `b [line-number]`
* list breakpoints: `breakpoint list`
* show assemble code of current line: `disassemble`
* read instruction in memory: `x/[bytes-count]i [function-address]`
* read registers: `register read [register-id]`, id example: `pc x1 x2`
* lookup function information: `image lookup -n "[imge-name]"`
* print data-structure size: `p sizeof([variable])`

