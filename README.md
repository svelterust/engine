# Beginners Guide to WebAssembly and Wasmer - Hello, world!

## Setup our new project by running the following commands (assumes we already have rust in our system)

https://www.youtube.com/watch?v=YkoEBKpyD7U
```sh
cargo new engine
cd engine
cargo add wasmer
cargo add wasmer-wasi
```

Might encounter an error if wasmer and wasmer-wasi version are not the same. In my case wasmer = "4.1.0" and wasmer-wasi = "3.1.1" did not show any error.

> IMPORTANT: Errors crept up while coding so in the end I changed both versions to "3.1". 

## Go to wasmer docs https://docs.rs/wasmer

Copy example code and replace default code in `src/main.rs`:
```rs
// src/main.rs
use wasmer::{Store, Module, Instance, Value, imports};

fn main() -> anyhow::Result<()> {
    let module_wat = r#"
    (module
      (type $t0 (func (param i32) (result i32)))
      (func $add_one (export "add_one") (type $t0) (param $p0 i32) (result i32)
        get_local $p0
        i32.const 1
        i32.add))
    "#;

    let mut store = Store::default();
    let module = Module::new(&store, &module_wat)?;
    // The module doesn't import anything, so we create an empty import object.
    let import_object = imports! {};
    let instance = Instance::new(&mut store, &module, &import_object)?;

    let add_one = instance.exports.get_function("add_one")?;
    let result = add_one.call(&mut store, &[Value::I32(42)])?;
    assert_eq!(result[0], Value::I32(43));

    Ok(())
}
```

```sh
cargo add anyhow
```

`anyhow` compiles a module written in webassembly text format (WAT). In this case the wat module assigned to `module_wat` variable has a function called `add_one` that accepts an integer `i32` and returns an integer `i32`. Basically adding `1` to the number passed in the `add_one` function.

We then create a global state for our webassembly using `Store::default()`
```rs
let mut store = Store::default();
```

The store represents all global state that can be manipulated by WebAssembly programs. It consists of the runtime representation of all instances of functions, tables, memories, and globals that have been allocated during the lifetime of the abstract machine.

The Store holds the engine (that is —amongst many things— used to compile the Wasm bytes into a valid module artifact).

Then create a module using the `store` and `module_wat`:
```rs
let module = Module::new(&store, &module_wat)?;
```

And finally create an instance of the module:
```rs
// The module doesn't import anything, so we create an empty import object.
let import_object = imports! {};
let instance = Instance::new(&mut store, &module, &import_object)?;
```

To use a function from the module we export the function first using `exports.get_function` on the module instance to get the desired function we wanted and assign it to a variable, in this case the variable is named `add_one` and the function we exported from `module_wat` is `add_one`:
```rs
let add_one = instance.exports.get_function("add_one")?;
```

Second to actually use the function use `call` method on the exported function, any returned value by the function will be assigned to our variable named `result`:
```rs
let result = add_one.call(&mut store, &[Value::I32(42)])?;
```

We passed the number `42` as the argument and we expect the value to increase by `1` making the returned value `43`. To assert the result we do:
```rs
assert_eq!(result[0], Value::I32(43));
```

To see the result when we run the rust app use `dbg!`:
```rs
dbg!(&result);
```

```sh
cargo run

> Finished dev [unoptimized + debuginfo] target(s) in 2m 41s
> Running `target/debug/engine`
> [src/main.rs:21] &result = [
    I32(43),
]
```

## Make our example wasm modular and reusable

We define a new `struct` named `Engine` (structs are like types in Typescript):
```rs
struct Engine {

}
```

### Making our rust app import a module from wat filepath

Let's add a funtion named `new` to our `struct` using `impl` (now our struct is like Object in JavaScript):
```rs
impl Engine {
    fn new() {
        
    }
}
```

Our code now would look like the following:
```rs
// src/main.rs
use wasmer::{Store, Module, Instance, Value, imports};

struct Engine {

}

impl Engine {
    fn new() {
        
    }
}

fn main() -> anyhow::Result<()> {
    // omitted code
}
```

Next is to move our `module_wat` code to our Engine function called `new`:
```rs
// src/main.rs
struct Engine {

}

impl Engine {
    fn new() {
        let module_wat = r#"
        (module
        (type $t0 (func (param i32) (result i32)))
        (func $add_one (export "add_one") (type $t0) (param $p0 i32) (result i32)
            get_local $p0
            i32.const 1
            i32.add))
        "#;

        let mut store = Store::default();
        let module = Module::new(&store, &module_wat)?;
        // The module doesn't import anything, so we create an empty import object.
        let import_object = imports! {};
        let instance = Instance::new(&mut store, &module, &import_object)?;

        let add_one = instance.exports.get_function("add_one")?;
        let result = add_one.call(&mut store, &[Value::I32(42)])?;
        dbg!(&result);
        assert_eq!(result[0], Value::I32(43));
    }
}

fn main() -> anyhow::Result<()> {
    // now we have nothing here

    Ok(())
}
```

We then write additional code that would allow our rust app to **read a module from file instead of hard coding it** using `Module::from_file`. The `from_file` method accepts two arguments, the first is the `engine` which is the `store` we have defined above and a filepath which is a `string`. For now we will name it as `file`:
```rs
// change
let module = Module::new(&store, &module_wat)?;
// to
let module = Module::from_file(&store, file)?;
```

Next add our required parameter called `file` as `&str` data type to our function:
```rs
fn new(file: &str) {
    // omitted code
}
```

Also remove or comment out the following code inside `fn new()` since we don't need it anymore:
```rs
let module_wat = r#"
    (module
    (type $t0 (func (param i32) (result i32)))
    (func $add_one (export "add_one") (type $t0) (param $p0 i32) (result i32)
        get_local $p0
        i32.const 1
        i32.add))
    "#;
```

And make sure that our function also returns a `Result`, we will use `anyhow::Result` for that so we import it first:
```rs
// src/main.rs
use anyhow::Result;

impl Engine {
    fn new(file: &str) -> Result<Self> {
        // omitted code
    }
}
```

We now also need to add two `fields` on our `Engine` struct which we will need for running webassembly and for storing the actual instance:
```rs
struct Engine {
    store: wasmer::Store,
    instance: wasmer::Instance
}
```

Then finally return `Self` in our `fn new()` at the very bottom of the function:
```rs
impl Engine {
    fn new(file: &str) -> Result<Self> {
        // omitted code
        dbg!(&result);
        assert_eq!(result[0], Value::I32(43));

        Ok(Self {
            store,
            instance
        })
    }
}
```

Our `fn new()` function now returns the store and instance of the given path of `wat` file. The next step is to separate the calls to the `wat` functions.

### Running functions from the imported wat file

Let's add a function named `run` to our `Engine` `impl`:
```rs
impl Engine {
    fn new(file: &str) -> Result<Self> {
        // omitted code
    }
    fn run() {
        // new run fn
    }
}
```

Then move the following code from `fn new()` to `fn run()`:
```rs
let add_one = instance.exports.get_function("add_one")?;
let result = add_one.call(&mut store, &[Value::I32(42)])?;
dbg!(&result);
assert_eq!(result[0], Value::I32(43));
```

Our `Engine` `impl` should now look like the following:
```rs
// src/main.rs
impl Engine {
    fn new(file: &str) -> Result<Self> {
        // let module_wat = r#"
        // (module
        // (type $t0 (func (param i32) (result i32)))
        // (func $add_one (export "add_one") (type $t0) (param $p0 i32) (result i32)
        //     get_local $p0
        //     i32.const 1
        //     i32.add))
        // "#;

        let mut store = Store::default();
        let module = Module::from_file(&store, file)?;
        // let module = Module::new(&store, &module_wat)?;
        // The module doesn't import anything, so we create an empty import object.
        let import_object = imports! {};
        let instance = Instance::new(&mut store, &module, &import_object)?;

        Ok(Self {
            store,
            instance
        })
    }
    fn run() {
        let add_one = instance.exports.get_function("add_one")?;
        let result = add_one.call(&mut store, &[Value::I32(42)])?;
        dbg!(&result);
        assert_eq!(result[0], Value::I32(43));
    }
}
```

Next is we need to pass `store` and `instance` to `fn run()`, we can do that by accepting `Self` as an argument to our `fn run()`:
```rs
fn run(&mut self) {
    // omitted code
}
```

Then make sure to use `self` when calling `store` and `instance`:
```rs
fn run(&mut self) {
    let add_one = self.instance.exports.get_function("add_one")?;
    let result = add_one.call(&mut self.store, &[Value::I32(42)])?;
    // omitted code
}
```

Same with `fn new()`, we also need to return a `Result` in `fn run()`, key thing to note here is that our `result` variable type is `Box<[Value]>`, so we need to explicitly make it as the returned `Result` type. The `Value` in this case is the one returned by `wasmer` which is `wasmer::Value`:
```rs
fn run(&mut self) -> Result<Box<[wasmer::Value]>> {
    // omitted code
    Ok(result)
}
```

Code is now like the following:
```rs
fn run(&mut self) -> Result<Box<[wasmer::Value]>> {
    let add_one = self.instance.exports.get_function("add_one")?;
    let result = add_one.call(&mut self.store, &[Value::I32(42)])?;
    // dbg!(&result);
    // assert_eq!(result[0], Value::I32(43));

    Ok(result)
}
```

Let's also change the function we are expecting to call from `add_one` to `main` and change the variable `add_one` to a more generic `function` instead:
```rs
// from
let add_one = self.instance.exports.get_function("add_one")?;
let result = add_one.call(&mut self.store, &[Value::I32(42)])?;
// to
let function = self.instance.exports.get_function("main")?;
let result = function.call(&mut self.store, &[Value::I32(42)])?;
```

The next step is to add `WASI` functionality as well. Head to [wasi docs](https://crates.io/crates/wasmer-wasi) and under Usage section copy the following:
```rs
// Create the `WasiEnv`.
let wasi_env = WasiState::new("command-name")
    .args(&["Gordon"])
    .finalize()?;

// Remove the part
.args(&["Gordon"])

// and change command-name to engine and pass `&mut store` to `.finalize()`
// so we get the following
let wasi_env = WasiState::new("engine").finalize(&mut store)?;
```

And paste it to our `fn new()` function just after the definition of `module` variable:
```rs
impl Engine {
    fn new(file: &str) -> Result<Self> {
        let mut store = Store::default();
        let module = Module::from_file(&store, file)?;
        // Create the `WasiEnv`.
        let wasi_env = WasiState::new("engine").finalize(&mut store)?;
    }
}
```

Also import `WasiState` from `wasmer_wasi`, our imports would now look like below:
```rs
// src/main.rs
use anyhow::Result;
use wasmer::{Store, Module, Instance, Value, imports};
use wasmer_wasi::WasiState;
```

Now we can use `wasi_env` to import the objects:
```rs
// change the following line
let import_object = imports! {};
// to
let import_object = wasi_env.import_object(&mut store, &module)?;
```

Next is to create a memory for `wasi` using the `instance` and pass the reference of the `memory` to `wasi_env`:
```rs
let instance = Instance::new(&mut store, &module, &import_object)?;

let memory = instance.exports.get.memory("memory")?;
wasi_env.data_mut(&mut store).set_memory(memory.clone());
```

We then want to be able to pass parameters to our `wasi` so we add another argument to `fn run()` function named `params` with type of `wasmer::Value`:
```rs
fn run(&mut self, params: &[wasmer::Value]) -> Result<Box<[wasmer::Value]>> {
    // omitted code
}
```

And use the `params` to the `function.call`:
```rs
// change
let result = function.call(&mut self.store, &[Value::I32(42)])?;
// to
let result = function.call(&mut self.store, params)?;
```

With the changes we made the `wasmer` import will become the following, removing `Value` and `imports`:
```rs
use wasmer::{Store, Module, Instance};
```

Our rust webassembly reader is now complete, we will try to run an actual webassembly file on the next section.

### Compiling rust for webassembly

We will create another rust project **inside our current rust project** that will be converted to webassembly file.
```sh
# print current directory
pwd
> projects/engine
# create new rust project with the --lib flag
cargo new hello --lib
# cd into the new hello folder
cd hello
```

Next we need to specify that the project is a library by adding `[lib]` with `cdylib` as `crate-type` to `Cargo.toml`. This is important so we could export our functions from webassembly file during webassembly runtime:
```toml
# hello/Cargo.toml

[package]
name = "hello"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[lib]
crate-type = ["cdylib"]

[dependencies]
```

Now that is done, we can go to `hello/src/lib.rs` and make hello world. Clear the contents of the file and replace it with the following:
```rs
fn main() {
    println!("Hello, world!");
}
```

After making the main function, we also want to specify the `no_mangle` attribute. By default Rust mangles the functions, it randomizes the names to make it optimal for the linker but this is not optimal for us developers (hoomans).
```rs
#[no_mangle]
fn main() {
    println!("Hello, world!");
}
```

That's it! We are now ready to build our hello world rust program into `wasm32-wasi`. Make sure to be inside the directory of `hellow-wasmer` before running cargo build:
```sh
cargo build --target=wasm32-wasi
```

If you don't have `wasm32-wasi` you can add it using rustup:
```sh
rustup target add wasm32-wasi
```

After building go back to our `Engine` folder and open `src/main.rs`:
```sh
# our current cirectory
pwd
> projects/engine/hellow-wasmer
# cd back one directory
cd ..
```

Then inside `fn main()` we need to get the first argument in the command line / terminal using `std::env::args()`:
```rs
fn main() -> anyhow::Result<()> {
    // we skip the first one since it's the name of our program
    // the next argument is what we need
    // then if there is no filepath specified, we throw an error message and panic
    let file = std::env::args().skip(1).next().expect("Path to wasm binary is expected.");

    Ok(())
}
```

Now we need to instantiate our `Engine` with the given filepath:
```rs
fn main() -> anyhow::Result<()> {
    let file = std::env::args().skip(1).next().expect("Path to wasm binary is expected.");
    let mut engine = Engine::new(&file)?;
    Ok(())
}
```

Finally run the `engine` by invoking the `run` function and passing an empty slice (&[]) as parameter:
```rs
fn main() -> anyhow::Result<()> {
    let file = std::env::args().skip(1).next().expect("Path to wasm binary is expected.");
    let mut engine = Engine::new(&file)?;
    engine.run(&[])?;
    Ok(())
}
```

To test simply run the rust app using cargo run along with the path to wasm file:
```sh
cargo run -- hello/target/wasm32-wasi/debug/hello.wasm
> Hello, world!
```
