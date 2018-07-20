---
layout: post
title: glitchcat — Creating CLI apps in Rust is super easy
---

[`glitchcat`](https://github.com/kuviman/glitchcat) is a cat-like program with glitch animation.

For example:

![Hello, world!]({{ "/assets/images/glitchcat/hello_world.gif" }})

It also works like magic in combination with `lolcat`:

![This is Magick!]({{ "/assets/images/glitchcat/this_is_magic.gif" }})

### Try it

To install `glitchcat`, you will first need [Rust compiler](https://rust-lang.org). Then just run

```sh
cargo install glitchcat
```

Or, to update, run

```sh
cargo install --force glitchcat
```

## Why?

In the process of ricing my Linux desktop, I have added several programs like `neofetch`, `lolcat` and `fortune` to my `~/.zshrc` to greet me every time I open a terminal. Now, with the `glitchcat` added to the mix, it looks like this:

![Rice]({{ "/assets/images/glitchcat/rice.gif" }})

To do same thing, you can put the following code in your `~/.zshrc` (or `~/.bashrc`, if you use `bash`):

```sh
# Dont show neofetch in VS Code integrated terminal
if [ "$TERM_PROGRAM" != "vscode" ]; then
    neofetch | lolcat
fi
fortune | glitchcat
```

*Actually, I lied, I dont use `fortune`. You know, you don't need fortune when you use Rust, right? ;)*

## Useful crates

Creating this program was actually pretty easy, with the help of the following crates:

#### - `structopt`

Almost every CLI app needs ability to be configured through command line arguments. `structopt` makes it very easy — just define a struct with everything you need, and `#[derive(StructOpt)]`:

```rust
#[derive(StructOpt)]
#[structopt(about = "cat-like program with glitch animation")]
struct Opt {
    #[structopt(
        short = "d",
        long = "duration",
        default_value = "1000",
        help = "Duration of animation in millis (or \"infinite\"/\"inf\")"
    )]
    duration: Duration,
    #[structopt(
        short = "a",
        long = "amount",
        default_value = "90",
        help = "Percentage of symbols glitched each animation step"
    )]
    amount: Percent,
    // ...
}
```

This will also automatically generate `--help` command for your application:

![glitchcat --help]({{ "/assets/images/glitchcat/glitchcat_help.png" }})

#### - `failure`

Another thing that everyone should handle is errors. Using `failure` crate you can `#[derive(Fail)]` for your custom error types easily:

```rust
#[derive(Fail, Debug)]
pub enum ParsePercentError {
    #[fail(display = "Value should be between 0 and 100")]
    TooBig,
    #[fail(display = "{}", _0)]
    ParseIntError(#[cause] <u8 as FromStr>::Err),
}

#[derive(Debug, Copy, Clone)]
pub struct Percent(u8);

impl FromStr for Percent {
    type Err = ParsePercentError;
    fn from_str(s: &str) -> Result<Self, ParsePercentError> {
        let value = match s.parse() {
            Ok(value) => value,
            Err(e) => return Err(ParsePercentError::ParseIntError(e)),
        };
        if value > 100 {
            Err(ParsePercentError::TooBig)
        } else {
            Ok(Percent(value))
        }
    }
}
```

![glitchcat --amount 100500]({{ "/assets/images/glitchcat/failure.png" }})

#### - `dialoguer` & `indicatif`

Sometimes you need to interact with user, like give him options to choose from. Or maybe you want to draw progress bars? `dialoguer` can do first thing and `indicatif` second. I didn't use them for `glitchcat`, but they can be useful for some other apps:

```rust
//! ```cargo
//! [dependencies]
//! dialoguer = "*"
//! indicatif = "*"
//! ```

extern crate dialoguer;
extern crate indicatif;

use dialoguer::Select;

fn main() {
    println!("To be or not to be?");
    let selections = &["Yes", "No"];

    let selection = Select::new()
        .default(0)
        .items(&selections[..])
        .interact()
        .unwrap();

    let bar = indicatif::ProgressBar::new(50);
    bar.set_style(indicatif::ProgressStyle::default_bar()
                      .template("Processing... {bar:30}"));
    for i in 0..50 {
        bar.inc(1);
        std::thread::sleep_ms(50);
    }
    bar.finish_and_clear();
    println!("What do you mean by {:?}?", selections[selection]);
    println!(
        "You are supposed to answer either {:?} or {:?}",
        "to be", "not to be"
    );
}
```

![To be or not to be]({{ "/assets/images/glitchcat/to_be.gif" }})

#### - `console`

Both `dialoguer` and `indicatif` are built upon the `console` crate. It allows you to easily add color to your output, or change cursor position inside terminal. That's basically all you need to implement a program like `glitchcat`:

```rust
//! ```cargo
//! [dependencies]
//! console = "*"
//! ```

extern crate console;

use console::{style, Term};

fn main() {
    let stdout = Term::buffered_stdout();
    stdout.write_line("Did you know?..");
    stdout.flush();

    let rust_is = [
        style("fast").green(),
        style("safe").green(),
        style("easy").red(), // Red here, since you need to read the Book first!
    ];
    let mut index = 0;

    loop {
        std::thread::sleep(std::time::Duration::from_millis(1000));
        stdout.clear_last_lines(1);
        stdout.write_line(&format!("Rust is {}!", rust_is[index]));
        stdout.flush();
        index = (index + 1) % rust_is.len();
    }
}
```

![Rust is]({{ "/assets/images/glitchcat/rust_is.gif" }})

## The End

This is it. Hopefully you find it useful (*I mean the article, `glitchcat` is pretty dumb*).

[Here's `glitchcat` project on GitHub](https://github.com/kuviman/glitchcat)