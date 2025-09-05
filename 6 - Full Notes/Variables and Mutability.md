[[Common Programming Concepts]]

We know what mutability and immutability is. When immutable, the value of a variable, once written, cannot be changed. 

If we try to do so - say hello to compile time errors!

If one part of our code operates on the assumption that a value will never change and another part of our code changes that value, it’s possible that the first part of the code won’t do what it was designed to do.

The cause of this kind of bug can be difficult to track down after the fact, especially when the second piece of code changes the value only _sometimes_. The Rust compiler guarantees that when you state that a value won’t change, it really won’t change, so you don’t have to keep track of it yourself. Your code is thus easier to reason through.

### Constants
Then there are *constants.* They are bound to a name, and *cannot* change. But, they are different from variables.

You declare constants using the `const` keyword instead of the `let` keyword, and the type of the value _must_ be annotated. Constants can be declared in any scope, including the global scope, which makes them useful for values that many parts of code need to know about. The last difference is that constants may be set only to a constant expression, not the result of a value that could only be computed at runtime.

*Rust’s naming convention for constants is to use all uppercase with underscores between words.*

They are valid for the entire time that the program runs, within the declared scope. 

### Shadowing

Declaring a new variable with the same name as the older variable is called *shadowing* .The second variable is what the compiler will see when you use the name of the variable. In effect, the second variable overshadows the first, taking any uses of the variable name to itself until either it itself is shadowed or the scope ends.

``` rust
fn main() {
    let x = 5;
    let x = x + 1;
    {
        let x = x * 2;
        println! {"Inner loop value of x is {x}"};
    }
    println! {"Outer loop value of x is {x}"};
}
```

Here, the inner loop will print `12` and the outer one does indeed print `6`. Do note, that we have used the `let` keyword everywhere, as that is what enables the shadowing to take place in the first place.

However, it is different from using the `mut` variable, as is we re-assign the variable without using let, we have a compile-time error. This reduces the chances of accidental re-naming. 