+++
date = "2016-05-14T13:06:00-04:00"
title = "Exploring Rust - Part 1 (Setup: Atom, Racer)"
description = "Started taking a look at what all the hype was about with Rust!"
tags = ["Programming", "Rust", "Compiler"]
topics = ["Blog", "Programming"]
comments=true
+++

## Exploring Rust
Everyone seems to be talking about Rust lately, so I'd thought I'd give it a shot and write something in it. The exploration begins with setting up the rust compiler (rustc), and setting up atom with various rust packages (linter, racer, and syntax highlighting).

### Environment Setup

#### Installing the Rust Compiler (rustc)

Fetch the latest stable rust binaries (As of writing this 1.17.0):
```bash
$ curl https://sh.rustup.rs -sSf | sh
```

Rust comes with a language package manager called Cargo, I will be using it to install a package called [racer](https://github.com/phildawes/racer) for auto completion in [atom](https://atom.io/).

#### Installing Racer
```bash
$ export PATH=$PATH:/home/$USER/.cargo/bin
$ cargo install racer
```

#### Atom Packages
* [language-rust](https://atom.io/packages/language-rust)
* [linter-rust](https://atom.io/packages/linter-rust)
* [racer](https://atom.io/packages/racer)

**Package Configuration:**
Racer requires ```RUST_SRC_PATH``` to be set to the location of your rust source, which can be downloaded and set like the following:
```bash
$ git clone https://github.com/rust-lang/rust.git
$ export RUST_SRC_PATH=/home/$USER/rust/src
```

"Ok, so I've opened a new ```.rs``` file and now its throwing errors at me!" - Make sure you've set the correct paths in Atom's racer package, so just ```Ctrl+,```, open the racer package, and set the paths. Mine are the following:

![Rust Racer Package Configuration](http://i.imgur.com/c6VnjVi.jpg)

And to test it out in your new rust source file, try out ```std::``` you should get some auto complete!

![Rust in Atom using Racer](http://i.imgur.com/tm2eeXU.jpg)

### Writing Rust

If you're used to higher level languages, rust might be slightly daunting - Then again it's probably not as bad as learning C/C++ the first time...

You can sometimes get helpful information using
```bash
$ rustc --explain EXXXX
```
Alternatively, the official [webified version](https://doc.rust-lang.org/error-index.html).

Happy rusting! Check out my other [rust articles](http://simonwillshire.com/public/tags/rust/)!

### Further Reading

* [Official Rust Documentation: Getting Started](http://doc.rust-lang.org/book/getting-started.html)
* [Rust Wiki](https://en.wikipedia.org/wiki/Rust_(programming_language)
