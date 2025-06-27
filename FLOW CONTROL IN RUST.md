[[Common Programming Concepts]]

Here is how an if statement works in rust - 
```rust
fn main() {
    let number = 3;

    if number < 5 {
        println!("condition was true");
    } else {
        println!("condition was false");
    }
}
```
Keep in mind, the condition in the loop's determining statement HAS to evaluate to a `bool` .
```rust
fn main() {
    let number = 3;

    if number {
        println!("number was three");
    }
}
```

This, will throw an error, as rust does not inherently typecast anything positive as true and 0 as false. 