[[Common Programming Concepts]]

Every value in Rust is of a certain _data type_, which tells Rust what kind of data is being specified so it knows how to work with that data. We’ll look at two data type subsets: scalar and compound.

Since rust is *statically typed*, we NEED to know the data types of all the variables in the program at *Compile time*. In cases when many types are possible, we must add a type annotation. If we don't there will be another compile time error. 

# SCALAR TYPES

There are 4. 
1. Integers
2. Floating-point
3. Booleans
4. characters

### Integer Type
A number without a fraction. 

| Length                 | Signed  | Unsigned |
| ---------------------- | ------- | -------- |
| 8-bit                  | `i8`    | `u8`     |
| 16-bit                 | `i16`   | `u16`    |
| 32-bit                 | `i32`   | `u32`    |
| 64-bit                 | `i64`   | `u64`    |
| 128-bit                | `i128`  | `u128`   |
| architecture dependent | `isize` | `usize`  |

Each variant can be signed (`i`)or unsigned (`u`).  If signed, it is always stored with the *2's compliment* method. 

#### Handling Overflows
Wrap around 0. gg.
 Or, to explicitly handle the possibility of overflow, you can use these families of methods provided by the standard library for primitive numeric types:
- Wrap in all modes with the `wrapping_*` methods, such as `wrapping_add`.
- Return the `None` value if there is overflow with the `checked_*` methods.
- Return the value and a Boolean indicating whether there was overflow with the `overflowing_*` methods.
- Saturate at the value’s minimum or maximum values with the `saturating_*` methods.

### Float Type
A number with a fraction.

Just 2 here, `f32` and `f64`.  Floating-point numbers are represented according to the IEEE-754 standard.

#### Numeric Operations
There are all the basic ones in rust. `Int` division does truncate to 0.
```rust
fn main() {
    // addition
    let sum = 5 + 10;

    // subtraction
    let difference = 95.5 - 4.3;

    // multiplication
    let product = 4 * 30;

    // division
    let quotient = 56.7 / 32.2;
    let truncated = -5 / 3; // Results in -1

    // remainder
    let remainder = 43 % 5;
}
```

### Boolean Type
There's 2 values, `true` (`1`) and `false` (`0`). They are 1 byte in size, and mostly used in conditionals. 

### Character Type
BRUH. COME ON NOW.

`''` are used to denote these. 

# COMPOUND TYPES

### Tuple Type
A generic way of grouping multiple types of variables together. It has a fixed length, so once declared, it cannot be re-sized. Like, at all. 
```rust
fn main() {
    let tup: (i32, f64, u8) = (500, 6.4, 1);
}
```

The variable `tup` will bind the entire tuple together. To get the individual values of the thing, we gotta do something like this - 
```rust
fn main() {
    let tup = (500, 6.4, 1);

    let (x, y, z) = tup;

    println!("The value of y is: {y}");
}
```

We can also access individual values of the tuple with the `.` operator.
```rust
fn main() {
    let mut x: (i32, i32) = (1, 2);
    x.0 = 0;
    x.1 += 5;
}
```
### Array Type
Every element is of the same type. 
```rust 
fn main() {
    let a = [1, 2, 3, 4, 5];
}
```

Arrays are useful when you want your data allocated on the stack, the same as the other types we have seen so far, rather than the heap. 
```rust
let a: [i32; 5] = [1, 2, 3, 4, 5];
```
To access, we can use indexing. 