[[Projects in Rust]]

### Pt 1 - User Input
This looks straightforward, but really isn't, lol. Here, we have to bring in the `io` library. This comes from the standard library, called as `std`.

By default, Rust has a set of items defined in the standard library that it brings into the scope of every program. This set is called the prelude. If a type you want to use isn’t in the prelude, you have to bring that type into scope explicitly with a `use` statement. Using the `std::io` library provides you with a number of useful features, including the ability to accept user input.

To accept a value, we gotta have a place to store it, and thus - welcome variables!

`let mut guess = String::new();`


The `let` statement helps create a variable. By default, everything in Rust is mutable. So, to make it not so, we have to explicitly state it - via the `mut` keyword. 

`let mut guess` will introduce a mutable variable named `guess`. The equal sign (`=`) tells Rust we want to bind something to the variable now. On the right of the equal sign is the value that `guess` is bound to, which is the result of calling `String::new`, a function that returns a new instance of a `String`. 
[`String`](https://doc.rust-lang.org/std/string/struct.String.html) is a string type provided by the standard library that is a growable, UTF-8 encoded bit of text.

The `::` syntax in the `::new` line indicates that `new` is an associated function of the `String` type. An _associated function_ is a function that’s implemented on a type, in this case `String`. This `new` function creates a new, empty string

Now, to actually RECEIVE the user input,  we gotta use the `stdin()` function. The stdin function returns an instance of `std::io::Stdin`, which is a type that represents a handle to the standard input for your terminal.

Next, the `real_line()` function gets the previously created `guess` string as an argument, thus, telling it to store anything that the user enters in the variable `guess`. 

Peep how we passed a `reference` to `guess`, by typing `&mut guess`. This is a feature of rust. Here, like variables, references are IMMUTABLE by default. So, we gotta let it know that it ain't, by the `mut` keyword. 

Now, there is default error handling in this bitch. How? By calling the `.expect()` function! 

As mentioned earlier, `read_line` puts whatever the user enters into the string we pass to it, but it also returns a `Result` value. [`Result`](https://doc.rust-lang.org/std/result/enum.Result.html) is an [_enumeration_](https://rust-book.cs.brown.edu/ch06-00-enums.html), often called an _enum_, which is a type that can be in one of multiple possible states. We call each possible state a _variant_.

`Result`’s variants are `Ok` and `Err`. The `Ok` variant indicates the operation was successful, and it contains the successfully generated value. The `Err` variant means the operation failed, and it contains information about how or why the operation failed.

So if `Err` is returned by the function being called, we have to have a way to tackle that eventuality. 

If this instance of `Result` is an `Ok` value, `expect` will take the return value that `Ok` is holding and return just that value to you so you can use it. In this case, that value is the number of bytes in the user’s input.

### Pt-2 Generating the Secret Number
As you'd have guessed, there is a random number generator dependency-thingy in rust. This is added to out build via the `cargo.toml` file. Add the necessary package that you want to use in your work under the `[dependencies]` tab. These are called as *crates* in rust. 

In the _Cargo.toml_ file, everything that follows a header is part of that section that continues until another section starts. In `[dependencies]` you tell Cargo which external crates your project depends on and which versions of those crates you require. 

When we include an external dependency, Cargo fetches the latest versions of everything that dependency needs from the _registry_, which is a copy of data from [Crates.io](https://crates.io/). Crates.io is where people in the Rust ecosystem post their open source Rust projects for others to use.

After updating the registry, Cargo checks the `[dependencies]` section and downloads any crates listed that aren’t already downloaded. In this case, although we only listed `rand` as a dependency, Cargo also grabbed other crates that `rand` depends on to work. After downloading the crates, Rust compiles them and then compiles the project with the dependencies available.

Cargo has a mechanism that ensures you can rebuild the same artifact every time you or anyone else builds your code: Cargo will use only the versions of the dependencies you specified until you indicate otherwise. 

For example, say that next week version 0.8.6 of the `rand` crate comes out, and that version contains an important bug fix, but it also contains a regression that will break your code. To handle this, Rust creates the _Cargo.lock_ file the first time you run `cargo build`, so we now have this in the _guessing_game_ directory.

When you build a project for the first time, Cargo figures out all the versions of the dependencies that fit the criteria and then writes them to the _Cargo.lock_ file. When you build your project in the future, Cargo will see that the _Cargo.lock_ file exists and will use the versions specified there rather than doing all the work of figuring out versions again. This lets you have a reproducible build automatically.

NOW, about the random numbers. We call the `rand::thread_rng()` function that gives a particular random number that is unique to the current thread of execution and is seeded by the OS. 

Next, we call the `gen_range` method on the random number generator. This takes a range of numbers as it's input, and gives back a random number. 

### Comparing the guess to the secret number.
Here, we'll have to make use of the `Ordering` enum, which is a part of the `std::cmp`
library. Be sure to bring this in the scope for the project. 

The `cmp` method compares two values and can be called on anything that can be compared. It takes a reference to whatever you want to compare with: here it’s comparing `guess` to `secret_number`. 

Then it returns a variant of the `Ordering` enum we brought into scope with the `use` statement. We use a [`match`](https://rust-book.cs.brown.edu/ch06-02-match.html) expression to decide what to do next based on which variant of `Ordering` was returned from the call to `cmp` with the values in `guess` and `secret_number`.

A `match` expression is made up of arms. Arms, consist of a pattern to match against, and should the input match the arm's pattern, a specific piece of code is to be run. Rust takes the value given to `match` and looks through each arm’s pattern in turn.