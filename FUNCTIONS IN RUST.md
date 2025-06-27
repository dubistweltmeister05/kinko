[[Common Programming Concepts]]

```rust
fn main() {
    println!("Hello, world!");

    another_function();
}

fn another_function() {
    println!("Another function.");
}
```

This is an example of a function in rust. Looks so much like C, doesn't it?!?! Rust doesn’t care where you define your functions, only that they’re defined somewhere in a scope that can be seen by the caller.

Of course they can have parameters
```rust
fn main() {
    another_function(5);
}

fn another_function(x: i32) {
    println!("The value of x is: {x}");
}
```

Here, we initialize an argument to the function, of the type `i32`. In the signature, we HAVE to declare the type of each parameter.  For multiple parameters, we separate them via commas - 
```rust
fn main() {
    print_labeled_measurement(5, 'h');
}

fn print_labeled_measurement(value: i32, unit_label: char) {
    println!("The measurement is: {value}{unit_label}");
}
```

### Statements and Expressions

- Statements are instructions that perform some action and do not return a value.
- Expressions evaluate to a resultant value.
Statements do not return values. Therefore, you can’t assign a `let` statement to another variable. 

Expressions do not include ending semicolons. If you add a semicolon to the end of an expression, you turn it into a statement, and it will then not return a value.

### Returning a value from a function

We don’t name return values, but we must declare their type after an arrow (`->`). The return value of the function is synonymous with the value of the final expression in the block of the body of a function.