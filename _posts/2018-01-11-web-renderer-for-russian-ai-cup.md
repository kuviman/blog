---
layout: post
title: Web renderer for Russian AI Cup
---

[Russian AI Cup](http://russianaicup.ru) &mdash; open artificial intelligence programming contest where you can test yourself writing a game strategy! It’s simple, clear and fun!
We welcome both novice programmers — students and pupils, as well as professionals. Writing your own strategy is very simple: basic programming skills are enough.

This competition was being held for the sixth time, and this time we made a game of the RTS game genre — players were controlling 500 vehicles of 5 different types at once. The task is to destroy the opponent!

![CodeWars 2017]({{ "/assets/images/raic/raic2018-screen.png" }})

My part was to implement the web renderer &mdash; the one you see on the site. There is also a technical renderer with schematic graphics used by participants for local testing.

## Problems with JavaScript

[Last year (2016)](http://2016.russianaicup.ru) we had a MOBA game, and the web renderer was written in JavaScript using the [three.js](https://threejs.org) framework.

![CodeWizards 2016]({{ "/assets/images/raic/raic2017-screen.png" }})

That time we had the fanciest graphics but there also were a lot of problems &mdash; it was running not smooth at all, the loading time was huge, it used a lot of RAM, was unwatchable on mobile devices, etc.
I did try to optimize it, and, although it became much smoother than it was during development it still is quite bad performance-wise.

## Starting with [Rust](https://www.rust-lang.org/en-US/)

JavaScript is not my main language, we mostly do Java at work, I did a lot of Python and C# at home, used C++ as a language for programming contests, and I have also tried and keep trying new languages from time to time.

I first heard about Rust about a year ago (no idea how I've missed it before), and almost immediately realized it is the language I want to use for everything from now on :)
I would say that Rust is somewhere between C++ and Haskell &mdash; it allows pretty high level abstractions that are safe while giving you a lot of control at the same time.
Rust's core feature is its [ownership system](https://doc.rust-lang.org/book/second-edition/ch04-00-understanding-ownership.html), which enables Rust to make memory safety guarantees without needing a garbage collector.

Here is a hello world program in rust:

```Rust
fn main() {
    println!("Hello, world!");
}
```

To run it:
1. [Install rust](https://www.rust-lang.org/en-US/install.html)
2. Create a new cargo project:
```sh
cargo new --bin hello-world
cd hello-world
```
3. Run:
```sh
cargo run
```

If you need to add a dependency, it is done easily in `Cargo.toml` file describing your project:

```toml
[package]
name = "hello-world"
version = "0.1.0"
authors = ["kuviman <kuviman@gmail.com>"]

[dependencies]
serde = "1"
```

There is an officially supported [IntelliJ Idea plugin for Rust](https://intellij-rust.github.io/), but you can use it in [other IDEs/editors](https://areweideyet.com/) too.

## [Rust](https://www.rust-lang.org/en-US/) + [Emscripten](https://kripken.github.io/emscripten-site/) &rarr; [WebAssembly](http://webassembly.org/)

I think it was also about a year ago when WebAssembly started appearing &mdash; but now it is supported on all major browsers. WebAssembly is a binary executable format for the web, and it is possible to compile C++ (and Rust, which is how I found out about it) to it using Emscripten. Emscripten is also capable of compiling to [asm.js](http://asmjs.org/), and it was used at first since WebAssembly was still unstable.

At the time I didn't know how well would Rust run in the browser, but that was an inspiring to try it out anyway &mdash; I wanted both to learn a language and to do something with graphics.

Running a WebAssembly version of hello world is a bit harder:

1. [Download Emscripten](http://kripken.github.io/emscripten-site/docs/getting_started/downloads.html) and follow installation instructions
2. Add Rust WebAssembly compilation target:
```sh
rustup target add wasm32-unknown-emscripten
```
3. Compile hello-world project to WebAssembly:
```sh
cargo build --target=wasm32-unknown-emscripten
```
4. Run via [Node.js](https://nodejs.org/en/)
```sh
cd target/wasm32-unknown-emscripten/debug
node hello-world.js
```

The js file contains Emscripten runtime and loads the WebAssembly.

Alternatively, to run in browser, create an HTML file like this:

```html
<html>
    <head>Hello, World!</head>
    <body>
        <script src="target/wasm32-unknown-emscripten/debug/hello-world.js"></script>
    </body>
</html>
```

And you should see `Hello, world!` in developers console.

There is also a [`cargo-web`](https://github.com/koute/cargo-web) cargo extension, which may make things easier, but I did not try it myself.

There is an issue with using Rust and Emscripten on Windows since Emscripten is called using a `.bat` file and there is a limit for `cmd.exe` on command line length, so using a lot of dependencies can result in exceeding this limit, which is annoying to come upon.

## Using OpenGL (WebGL)

First googling around suggested using [`glium`](https://crates.io/crates/glium) & [`glutin`](https://crates.io/crates/glutin). This is not as high-level as three.js, but was easy to start with. The problem was that it worked on desktop, but didn't compile into any Emscripten target. So instead [OpenGL](https://crates.io/crates/gl) was used directly, and it did work. The way it works is that Emscripten translates calls to OpenGL into calls to WebGL. Maybe now as some time has passed you can use `glutin` with `glium` (or with [`gfx`](https://crates.io/crates/gfx), which is not so tied with OpenGL, but rather abstracts over all the graphics APIs). Anyway, using OpenGL as it is is unsafe so a safe wrapper crate like these was created.

It was a little hard to do rendering with pure OpenGL, but it was fun to learn some new things anyway. This year we had only static models, so it was only needed to parse obj, draw particles and implement simple shadows.

[Instancing](https://www.khronos.org/opengl/wiki/Vertex_Rendering#Instancing) was used to speed up rendering, which required that your device supported [WebGL extension](https://www.khronos.org/registry/webgl/extensions/ANGLE_instanced_arrays) for it, but it is supported on most devices. There was a problem with [depth textures](https://www.khronos.org/registry/webgl/extensions/WEBGL_depth_texture) on some devices (not supported on some, rendering with artifacts on others), so regular textures were used for shadow maps.

## Interacting with JavaScript

Any Emscripten API function can be called as if you were writing C code. Interop with C is pretty easy in Rust.

```Rust
use std::ffi::CString;
use std::os::raw::c_char;

extern "C" {
    pub fn emscripten_run_script(script: *const c_char);
}

fn run_script(script: &str) {
    let script = CString::new(script).unwrap();
    unsafe { emscripten_run_script(script.as_ptr()); }
}

fn main() {
    run_script("console.log(\"Hello, world!\"");
}
```

There is also a macro system in rust that enables DSLs inside Rust:

```Rust
fn main() {
    let message = "Hello, world!";
    run_js! {
        console.log(@message);
    }
}
```

Several crates for Rust on web have appeared since the project started, like [`stdweb`](https://crates.io/crates/stdweb), which is probably a better solution.

## [JSON](https://en.wikipedia.org/wiki/JSON)

JSON format is used in game log, and to interact with JavaScript. There is a great serialization/deserialization library for rust called [`serde`](https://crates.io/crates/serde), which together with [`serde_json`](https://crates.io/crates/serde_json) provides easy deserialization of JSON into typed Rust objects, like:

```Rust
#[derive(Serialize, Deserialize)]
struct Person {
    name: String,
    age: u8,
    phones: Vec<String>,
}

fn main() {
    let data = r#"{
                    "name": "John Doe",
                    "age": 43,
                    "phones": [
                      "+44 1234567",
                      "+44 2345678"
                    ]
                  }"#;
    let p: Person = serde_json::from_str(data)
        .expect("Failed to parse JSON");
}
```

## Custom derive in Rust

The above example uses custom derive feature in Rust, which is basically code generation. There is no reflection in Rust, so the `Deserialize` custom derive implements `serde::Deserialize` trait on `Person`, which you could otherwise write yourself, but there is only boilerplate in such code.

Custom derive is also used in the renderer for OpenGL vertex data structures, resources, and settings:

```Rust
#[derive(Resources)]
pub struct FactoryResources {
    #[path = "assets/facilities/Factory.obj"]
    factory: obj::Model,
    #[path = "assets/facilities/FactoryGray.png"]
    factory_neutral: opengl::Texture,
    #[path = "assets/facilities/FactoryOrange.png"]
    factory_1: opengl::Texture,
    #[path = "assets/facilities/FactoryBlue.png"]
    factory_2: opengl::Texture,
}

#[derive(Settings)]
pub struct CodeWars2017Settings {
    #[setting(name = "Time scale", default = "1.0", range = "0.0 .. 4.0")]
    pub time_scale: f64,
    #[setting(name = "Minimap", default = "true")]
    pub draw_minimap: bool,
    #[setting(name = "Shadows", default = "true")]
    pub shadows_enabled: bool,
    #[setting(name = "Fog of war", default = "true")]
    pub fog_enabled: bool,
}
```

## Handling errors

Rust does not have exceptions, instead it has `Result` for recoverable errors, which is a type that is either a success with some value, or an error with error description. This is a more type-safe version of C error codes.

For example, result of OpenGL shader compilation is either a compiled shader or an compilation error string.

```Rust
impl Shader {
    pub fn compile(source: &str, typ: ShaderType) -> Result<Self, String> {
        // Pseudo code
        let shader = glCreateShader();
        glCompileShader();
        if not_compiled {
            return Err(compilation_log);
        }
        Ok(shader)
    }
}
```

There are also unrecoverable errors in Rust that are called "panics". In case your program panics, it just shows the error message with stacktrace and exits. You can though setup a panic hook to display a fancy error message so that opening dev console in browser is not necessary:

```Rust
fn panic_hook(info: &std::panic::PanicInfo) {
    let error: String = if let Some(error) = info.payload().downcast_ref::<String>() {
        error.clone()
    } else if let Some(error) = info.payload().downcast_ref::<&str>() {
        error.to_string()
    } else {
        String::from("Something went wrong")
    };
    run_js! {
        show_error(@error);
    };
}
std::panic::set_hook(Box::new(panic_hook));
```

![Error]({{ "/assets/images/raic/raic2018-error.png" }})

There are also some crates that allow easily getting stack trace of an error and error chaining like [`failure`](https://crates.io/crates/failure), but they were not used.

One issue there was with WebAssembly is that it is the only place where [undefined behaviour](https://github.com/rust-lang/rust/issues/10184) in Rust was seen &mdash; when casting negative floats to an unsigned integer, which resulted in a WebAssembly exception instead of Rust error or some random number like when running natively, and that produced another error in Rust later since the exception was not catched.

## Running natively

Although originally the renderer was being written to run in browser it is possible to compile and run it as a native application. Most of the code is reused, but some parts have to be rewritten, like reading files instead of sending requests, or creating the window instead of a canvas element. This can be easily done with conditional compilation:

```Rust
impl GLContext {
    #[cfg(target_os = "emscripten")]
    pub fn new() -> Result<Self, GLContextCreationError> {
        // WebGL
    }
    #[cfg(not(target_os = "emscripten"))]
    pub fn new(window: &Window) -> Result<Self, GLContextCreationError> {
        // OpenGL
    }
}
```

Right now the renderer does work natively, the only unimplemented part is the UI, since HTML is used for it while in browser.

## Conclusion

Rust was surprisingly very easy to start with &mdash; [The Book](https://doc.rust-lang.org/book/) is really good, the package manager (cargo) is super easy to use, compiling to WebAssembly is as easy as running natively once you get Emscripten. The Rust Community is also working on a new [`wasm32-unknown-unknown`](https://www.hellorust.com/news/native-wasm-target.html) target that does not require Emscripten to compile your code.

Overall, using Rust for the renderer was much more pleasant than implementing it in JavaScript, and it looks like the results are indeed much better. This year game logs were bigger than before, and using WebAssembly really helped to process them while rendering at the same time.