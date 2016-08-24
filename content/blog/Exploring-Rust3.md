+++
date = "2016-07-24T14:54:00-04:00"
title = "Exploring Rust - Part 3: Writing a Programming Language Parser (Nom)"
description = "Taking a look at writing a programming language parser using Nom"
tags = ["Programming", "Rust", "Nom"]
topics = ["Blog", "Programming"]
comments=true
+++

## Rust

I assume you have already installed it, if not check out the [first part](http://simonwillshire.com/blog/Exploring-Rust/) of the series.

## Writing Your Own Programming Language parser

So, recently I've been looking into improving/extending a [Brainfuck]('https://en.wikipedia.org/wiki/Brainfuck') interpreter that I wrote a while back to pick up a little more Rust and eventually write a full compiler. For the purpose of this article, I will be catering the parser towards capturing tokens that pertain to Brainfuck or an extended variant of it, however the concepts are largely the same.

### Nom?

Check out the official [github repo]('https://github.com/Geal/nom') to get some background on the parser. It is different from other parsers in that it captures smaller variants of expressions which build on each other. For example, you could capture an integer and use that integer format in other expressions later.

Alright, so let's get nom installed, create a new Rust project, and throw in

```yaml
[dependencies]
nom = "*"
```

Create a new ```main.rs```, and a subfoldered ```parser/lib.rs``` from the project's root:

***[main.rs]('https://github.com/Tiggilyboo/Backtick/blob/master/src/main.rs')***
```rust
#[macro_use]
extern crate nom;
use std::collections::HashSet;
mod parser;

fn main(){
    let p = b"
        @0,^in
        @1^out
        ^copy @0:1 !`
            @in[->+>+<2]
            @out.
            ~
        `
        @in!copy";
    let tokens = parser::parse(p);
    ...
}
```

***[parser/lib.rs]('https://github.com/Tiggilyboo/Backtick/blob/master/src/parser.rs')***
```rust
use nom::{IResult, digit, alphanumeric, multispace, not_line_ending};
use std::str;
use std::str::FromStr;

#[derive(Debug, Clone)]
pub enum Token {
    Address(u16),
    Label(String),
    Comment(bool),
    Loop(Vec<Token>),
    Multiplier((u8, u16)),
    Operator(u8),
    Set(u16),
    ...
}

named!(number<u16>,
  map_res!(
    map_res!(
      digit,
      str::from_utf8
    ),
    FromStr::from_str
  )
);

named!(string<String>,
    chain!(
        a: alphanumeric,
        || String::from_utf8(a.to_vec()).unwrap()
    )
);

named!(eol,
    chain!(
        alt!(tag!("\n") | tag!("\r\n") | tag!("\u{2028}") | tag!("\u{2029}")),
        || { &b""[..] }
    )
);

named!(blanks,
    chain!(
        many0!(alt!(multispace | eol)),
        || { &b""[..] }
    )
);
...
```

So what's going on here? ```Token``` allows us to store various operations in our captured language. I've shortened the list to the essential operations:

* Address: Sets the current position of the pointer to the supplied number.
* Label: Looks for a prefixed character and captures the string afterward.
* Comment: Ignores the captured string to comment in code.
* Loop: Captures a typical ```[``` and ```]``` in normal BF, as well as any extended operations within.
* Multiplier: Captures a BF operation and a number afterwards supplying the number of times to invoke the operation.
* Operator: A single BF operator (no multiplier), and any other extended ```Token```.
* Set: Assigns the current memory value to the captured number

Now that we have the basic Tokens laid out, let's run through capturing some of them in Nom. The above snippet is a base, that can be used for generic number and string matching (with or without blands/eols) and is used throughout the other Token macros.

Alright, so let's start with something easy, ignoring areas of input (like Comments):

```rust
named!(blank<Token>,
    chain!(
        alt!(multispace | eol),
        || Token::Comment(false)
    )
);
```

* ```!chain``` allows for multiple macros to be run in succession using the ```~``` character to join them, and returns the capture ```||``` which in this example is the num ```Comment``` with false as its value.
* ```!alt``` captures left to right the first macro that applies to the input bytes, so in this case any spaces or end of line characters (\r and \n).
Alright, now on to something more complex, capturing an Address Token:

```rust
named!(address<Token>,
    chain!(
        n: preceded!(tag!("@"), number),
        || Token::Address(n)
    )
);
```

* Using our same ```!chain``` macro, we capture any string of bytes that has a prefix of ```@``` with a number after (Building on our base snippet macro ```number``` from before.)
* We then store the captured expression ```n``` into our enum ```Token::Address``` and that's that.

So to go further ahead we need to explain what we want to capture first, in this soon-to-be-less-metaphorical language, we want to define functions, which I call *backtick expressions*, lets examine the format:

```
^copy @0:1 !`
  [->+>+<2]
  ~
`
```

* ```^copy```: Defines a label 'copy' that can be used later to call this *backtick expression*
* ```@0:1```: Assigns this backtick expression memory allocation starting at position 0 to position 1 (ie. allocate 1 byte).
* ``` !`<expression>` ```: Denotes that this expression is a function (can be executed instead of assigned, more on this next), and it executes the captured *<expression>*.
* ```[->+>+<2]```: Basically the same as standard BF, however we make use of a multiplier ```<2``` which executes the ```<``` operator twice.
* ```~```: Added operator to break the current function, the position pointer is not affected. Useful for within loops if you wish to exit.

So, now that we've defined a function, let's capture the macro for it:

```rust
named!(expression<Token>,
    alt!(blank | comment | label | address | multiplier | brackets |
        operator | condition | execute | set)
);

named!(backtick_expression<Vec<Token> >,
    delimited!(char!('`'), many0!(expression), char!('`'))
);

named!(backtick<Token>,
    chain!(
        l: preceded!(tag!("^"), string) ~
        blanks? ~
        s: opt!(preceded!(tag!("@"), number)) ~ blanks? ~
        o: opt!(preceded!(tag!(":"), number)) ~ blanks? ~
        c: preceded!(tag!("!"), backtick_expression),
        || Token::Function((l, s, o, c))
    )
);
```

*Whaaaa?* - Right so, this is where the power of Nom comes in where we can really start stacking macros. First of all, we capture an ```expression```, the token returned from this, could be ```blank```, ```comment```, ```address``` etc. (To see full implementation, the [github repo is your friend]('https://github.com/Tiggilyboo/Backtick')). Next up, that expression can be used 0 or more multiple times (```many0!```), between backtick characters ``` ` ```. Now, putting this all together:

* Check for a label ```^```, and capture a string after it,
* Check for any optional ```blanks?```,
* Check for any optional starting address or ending address ```@<num>:<num>``` with optional ```blanks?```
* Find a function declaration ``` !`<backtick_expression>` ``` (Which crams in everything we explained above).

All of that information can be stored within a single ```Token``` (which stores a bunch of other Tokens). It is important to note that these macro's are order sensitive, so be careful that you capture comments before expressions, and functions before loops and what have you so that you can process your parsed tokens afterward in the proper order!

If you want to read more on the compiler I'm writing (coined `backtick`) check out the [repo]('https://github.com/Tiggilyboo/Backtick/').

I'll try and keep these **Exploring Rust** articles more connected in the future, next in the series I'll be writing about processing all our Nom'd tokens into a tree structure (AST) to be generate LLVM.

Until next time! Ciao
