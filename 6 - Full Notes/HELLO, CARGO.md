[[Getting started with RUST]]

This is the default RUST package manager and build system. It builds the code, but also installs dependencies, and builds those as well.

This, basically makes life easier when we write complex stuff that will blow our minds eventually. 

When we use the cargo manager, we get two files, a `.toml` file and a `src` directory.

`TOML` stands for (`Tom's Obvious, Minimal Language`), and is the configuration format for cargo. 

Cargo expects your source files to live inside the _src_ directory. The top-level project directory is just for README files, license information, configuration files, and anything else not related to your code. Using Cargo helps you organize your projects. There’s a place for everything, and everything is in its place.

To build using `cargo`, use the `cargo build` command and run, it's simply `cargo run`. 

`cargo check` is another way of checking if the project will run or not, and it doesn't generate the binaries, so that's a cleaner way of checking if everything will run or not.

Also, if something compiles in rust, it WILL run.