---
title: "Ergonomic error handling with Rust"
date: 2021-03-28
draft: false
image: https://www.kolpaper.com/wp-content/uploads/2020/07/Vaporwave-Error-Wallpaper.jpg
---

Prior to learning Rust, I never knew error handling was actually a big deal. It never struck me
that there are alternative ways to handling errors, that error messages should be as informative
as possible, or even that you should take time to write robust error handling logic in your code.
It's obvious in retrospect, but in Python-land, there isn't as big a chorus on error handling.

Rust's story is still developing and as a result there are conversations about every aspect of the
language. With a strong emphasis on making developing in Rust enjoyable, there is plenty of healthy debate
going on. Error handling is an ongoing and important discussion in the community. The language attempts
to make error handling ergonomic and I think they've done a great job of it so far. As I started to read
more about it, I realised there was so much I had never considered. Writing good errors and having robust
error handling logic are core features of writing good software. Alongside this, they contribute
immensely to developer experience. The central theme threaded throughout this post is this:

> Error handling is about communication

We strive to write robust code with good error handling logic and informative errors so that we can
accurately describe what happened and how we resolved it. Without further ado, let's jump into the world
of error handling.

## Why you should learn about distributed systems

[Why you should learn about distributed systems]: #why-you-should-learn-about-distributed-systems

In distributed systems, if there is anything you can be certain of, it is that your system
*will* fail. Failure is normal. These systems must be built to be fault-tolerant. In learning more
about them, one idea really stuck with me and shaped my thinking: build for failure. This thinking
is applicable in most aspects of software development. How many times have you used software and it
returns the world's most unhelpful error messages? It once took me about a full working day to debug
an error that was fixed by using a jpg image instead of a png. The error message had absolutely nothing
to do with the image format and so I wasted countless hours researching and debugging. The application
was not built to handle (reasonable) failures.

Error handling is such an important part of development precisely because it allows us to handle failure.
It improves our ability to reason about our software; we understand the myriad ways it may fail. When
we encounter failures, great error messages help us diagnose problems efficiently. This in turn boosts
productivity and leads to an improved developer experience.

## Shifting paradigms

[Shifting paradigms]: #shifting-paradigms

Languages like Rust and Go have returned to ways of ancient past by returning errors instead of using
exceptions. Being accustomed to exceptions, I did not understand the benefits of returning errors.
Through researching it, I came across justifications for its use:

* Returning errors is explicit. In Python, any function *could* throw an exception. Short of reading
  the source code or the documentation, you're left for dead. If errors are returned, e.g
  `obj, err = func()`, they are pushed onto the developer explicitly. She is at least aware that the
  function may result in an error. It is still left to her to decide to handle or ignore it but she
  does so consciously. Without the explicitness, it is too easy to miss areas of code that should have
  exception handling, purely because you do not even know exceptions are raised. In fairness, statically
  typed languages with exceptions can solve this problem. For example, Java function signatures declare
  the exceptions they throw and the compiler enforces exceptions. Fortunately for all Java developers,
  there is a loophole out of that requirement; one that is used all
  [too often](https://www.overops.com/blog/ignore-checked-exceptions-all-the-cool-devs-are-doing-it-based-on-600000-java-projects/).
* Exceptions are often misused. They should only be used in *exception*-al cases. This is one of those
  dogmatic phrases in tech but there is some merit behind it. Exceptions do not need to be raised in
  situations where errors are expected. For example, if we need to read a file and know that there is
  a possibility that the file may not exist, an exception should not be raised. We expected this and
  should write our code to handle that situation or propagate that information back to the caller. It's
  not an exceptional situation. However, if we need to read a file and expect the file to be there and
  it is not, an exception can be raised. It's a situation we did not expect and possibly one we do not
  know how to handle. This was the original thinking behind exceptions - in most circumstances, we have
  logic to handle known, expected failure cases but when we run into truly exceptional situations, we
  raise exceptions. In reality, exceptions are used freely. One horror story I came across was from
  a developer who complained about an HTTP client library he had tried using. It threw exceptions for
  every single non 2xx and 3xx response. That's a great example of misusing exceptions.
* [Pokemon exception handling]. In Java and Python, it's possible to catch all exceptions by using syntax
  like `catch Exception` since all exceptions subclass `Exception`. This makes debugging failure modes
  extremely difficult. If your code starts producing weird outputs, it's impossible to tell because errors
  are caught silently by the catch all statement. In reality, this is the fault of the developer but
  developers are lazy and will find loopholes if the system allows.

After reading that list, you'd be inclined to think that returning errors is hands down the better option.
Returning errors has a critical flaw: it is *easy* to ignore them. For example in Go, you can write
`value, _ := func()`. In the event that `func` errors, `value` will be left in an invalid state but
execution will continue uninhibited. With exceptions, a key benefit is that program will crash immediately.
Returning errors requires significant discipline from developers, who as we mentioned before, are lazy.

Rust takes returning errors to the next logical level by enforcing errors are handled. It takes inspiration
from the [ML family of languages]. These languages have a strong type system, embedding the notion of
a fallible computation into the type system in a unique way.

## How Rust does error handling

[How Rust does error handling]: #how-rust-does-error-handling

### The Result type

In the ML family of languages, one of the common constructs is the `Result` type. A `Result` is an enum
that has two possible states: the value of the computation or an error. To get the value, you have to
unwrap `Result`. The compiler enforces that `Result` is unwrapped before it is *used* anywhere else.
For example, if we have a function `do_something(s: String)` and our input variable is currently a
`Result<String>`, we have to unwrap it to get the `String` in order to pass it into the function.
In the event `Result` contains an error, we have to handle it. This enforces the handling of errors.
There is no way to circumvent it due to it being embedded in the type system. This is why error handling
in Rust is great. Let's take a look at basic error handling.

```Rust
// This is the result type in the standard library. It either contains
// the value of the computation Ok(T) where T is the value or an error
// Err(E) where E is the error
enum Result<T, E> {
   Ok(T),
   Err(E),
}
```

```Rust
let mut f = File::create("ferris.txt");  // Returns Result<File>

// To unwrap we match on the enum variants
let mut file = match f {
  Ok(f) => f,
  Err(e) => panic!("File could not be created")
};

match file.write_all("Hi Ferris") {
  Ok(_) => {},
  Err(e) => panic!("Could not write to file")
}
```

Rust has several convenience methods/syntax such as `unwrap`, `expect`, `?` which help to remove some
of the boilerplate. We will discuss these later.

### Panicking

In Rust, irrecoverable errors are signaled using the `panic!` macro. When a panic is invoked, the
developer is essentially saying, "program execution cannot continue any further after encountering
this error". Panicking is a terminal state; the program crashes as a result. It is normally used when
a bug is encountered. They are meant to be infrequent, with standard error handling taking care of
most cases. We've already seen panic being used in Rust in the above example.

```Rust
match file.write_all("Hi Ferris") {
  Ok(_) => {},
  Err(e) => panic!("Could not write to file") // If this branch executes, program crashes
}
```

### Bubbling errors

In many cases, we do not want an error to be handled at the location it is generated. Instead, we
would prefer to have that error be handled by the caller of the function, giving it the power to decide
how to proceed. In programming parlance, we want to "bubble" the error to the caller. In Rust, we can
achieve this using matching.

```Rust
// () is the unit type and is used when there is no meaningful
// value to return
fn init() -> Result<(), io::Error> {
  let mut file = match File::create("ferris.txt") {
    Ok(f) => f,
    Err(e) => return Err(e) // returns an error to the caller
  };

  match file.write_all("Hi Ferris") {
    Ok(_) => return Ok(()),
    Err(e) => return Err(e)
  }
}

```

```Rust
// The executable code
// Caller of function gets error and then panics on failure
match init() {
  Ok(_) => {},
  Err(_) => panic!("Could not initialise")
}
```

## Making it ergonomic

[Making it ergonomic]: #making-it-ergonomic

As I mentioned earlier, Rust has a number of convenience methods/syntax to reduce boilerplate. There
are three that are commonly used: `unwrap`, `expect` and `?`.

### Try

When bubbling errors, you can imagine that writing the above match statement becomes cumbersome. It's
boilerplate that the language can handle for you. A similar problem exists in Go. If you ask Gophers
what line of code they write the most, they'll answer: `if err != nil {}`. The Rust language designers
took care of matching boilerplate by introducing the `?` syntax. This replaced the `try!` macro as a
more convenient syntax. Its purpose is to automatically bubble the error to the caller on an error
occuring.

```Rust
 // Before we would write code like this
fn init() -> Result<(), io::Error> {
  let mut file = match File::create("ferris.txt") {
    Ok(f) => f,
    Err(e) => return Err(e)
  };

  match file.write_all("Hi Ferris") {
    Ok(_) => Ok(()),
    Err(e) => return Err(e)
  };
}
```

```Rust
// With the `?` operator, we can write it as
fn init() -> Result<(), io::Error> {
  let mut file = File::create("ferris.txt")?;
  file.write_all("Hi Ferris")?;
}
```

As you can see from the above example, our code is significantly less verbose while being functionally
identical. This is one of those small quality of life improvements that makes a world of difference when
writing Rust. It highlights the commitment to a friendly developer experience.

### unwrap

There are often scenarios where you want to opt out of error handling. This may be when you're prototyping
and don't want to go through the effort of setting up robust error handling or when you know that a function
won't fail (e.g if you need to read a file that you know will always exist). To get out of it, you can
use the `unwrap()` method. `unwrap` returns the `Ok` variant with it's value if the computation succeeds
or will panic on error.

```Rust

// Notice how we don't need the return type anymore
fn init() {
  // We use `unwrap()` to panic if it fails otherwise execution continues
  let mut file = File::create("ferris.txt").unwrap();
  file.write_all("Hi Ferris").unwrap();
}
```

If we try create a file in a location we do not have permission to access, the code will panic, causing
the program to crash. An example of a panic message

```
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Os {
code: 13, kind: PermissionDenied, message: "Permission denied" }',
src/main.rs:4:36
```

Notice how Rust does not give us a get out of jail free card with error handling. Either we write error
handling logic or we accept our code will panic and subsequently crash (most likely unknowingly to the
developer). Extensive `unwrap`ing is not recommended for production code. There are cases where it is
permissible (as with everything), for example if we can guarantee it won't fail or we want execution
to panic at that location. One issue with `unwrap` is that error messages can be uninformative. We can
do one better by using `expect`.

### expect

The `expect()` method is identical to `unwrap()` but it allows you to set an error message. This conveys
your intent and makes debugging easier.

```Rust

fn init() {
  // Write to a location we do not have permission
  let mut file = match File::create("/var/ferris.txt")
    .expect("Could not create file `ferris.txt`");

  file.write_all("Hi Ferris").expect("Could not write to file");
}
```

This generates the message

```
thread 'main' panicked at 'Could not create file `ferris.txt`: Os {
code: 13, kind: PermissionDenied, message: "Permission denied" }',
src/main.rs:4:36
```

As we can see, Rust has a strong focus on making error handling a frictionless experience for the
developer. There are other functions that I have not gone through here: `map_err`, `map_or_else`,
`unwrap_or`, `unwrap_or_else` and many more. If you're new to Rust, I encourage you to give them a
look. I certainly will as I improve the quality of my error handling.

## Making it informative

[Making it informative]: #making-it-informative

Writing good errors means making them highly informative. Without good error messages, debugging
is significantly more difficult. Either that or you've learnt how to read extremely cryptic
messages. C++ programmers, you're *obviously* suffering from Stockholm syndrome given that you
eat these crazy error messages for breakfast 😆.

```
rtmap.cpp: In function `int main()':
rtmap.cpp:19: invalid conversion from `int' to `
   std::_Rb_tree_node<std::pair<const int, double> >*'
rtmap.cpp:19:   initializing argument 1 of `std::_Rb_tree_iterator<_Val, _Ref,
   _Ptr>::_Rb_tree_iterator(std::_Rb_tree_node<_Val>*) [with _Val =
   std::pair<const int, double>, _Ref = std::pair<const int, double>&, _Ptr =
   std::pair<const int, double>*]'
rtmap.cpp:20: invalid conversion from `int' to `
   std::_Rb_tree_node<std::pair<const int, double> >*'
rtmap.cpp:20:   initializing argument 1 of `std::_Rb_tree_iterator<_Val, _Ref,
   _Ptr>::_Rb_tree_iterator(std::_Rb_tree_node<_Val>*) [with _Val =
   std::pair<const int, double>, _Ref = std::pair<const int, double>&, _Ptr =
   std::pair<const int, double>*]'
E:/GCC3/include/c++/3.2/bits/stl_tree.h: In member function `void
   std::_Rb_tree<_Key, _Val, _KeyOfValue, _Compare, _Alloc>::insert_unique(_II,
    _II) [with _InputIterator = int, _Key = int, _Val = std::pair<const int,
   double>, _KeyOfValue = std::_Select1st<std::pair<const int, double> >,
   _Compare = std::less<int>, _Alloc = std::allocator<std::pair<const int,
   double> >]':
E:/GCC3/include/c++/3.2/bits/stl_map.h:272:   instantiated from `void std::map<_
Key, _Tp, _Compare, _Alloc>::insert(_InputIterator, _InputIterator) [with _Input
Iterator = int, _Key = int, _Tp = double, _Compare = std::less<int>, _Alloc = st
d::allocator<std::pair<const int, double> >]'
rtmap.cpp:21:   instantiated from here
E:/GCC3/include/c++/3.2/bits/stl_tree.h:1161: invalid type argument of `unary *
```

To reiterate, error handling and error reporting is about communication. By providing highly informative
errors, we paint the full picture of what has happened. We want to guide the developer directly to the
problem. Rust has made good error reporting a core emphasis of the language and the larger ecosystem.
Arguably, unintentionally.

{{< tweet 1303107711928643584 >}}

The Rust compiler really is a gold standard in error reporting. There's still work to do but it is one
of the most human friendly pieces of software I have used. As an example

{{< figure src="/images/rust-error-handling/rounded-compiler.png" >}}

I quite like how this tweet puts it :)

{{< tweet 1348669062528774148 >}}

Focusing on good error messages has permeated throughout the community. There's even the [Error Handling Project Group]
if you weren't convinced how committed the language designers are to getting this right. There are a
number of techniques we can use to make our errors more informative. Along the way, we will discuss the
crates that can help.

### What are we writing?

The first question we have to ask is

> Are we writing an application or a library?

I had never considered that your approach to structuring error handling is different depending on
whether you are writing an application or library. This [article](https://nick.groenen.me/posts/rust-error-handling/)
illuminated this difference to me. I encourage you to go read it. I will mention the points I found
particularly insightful with some insight of my own.

* As far as possible, libraries should use meaningful custom error types. This has several benefits:
  * Applications can easily differentiate between various errors. A simple example are the
    various [IO errors](https://doc.rust-lang.org/std/io/enum.ErrorKind.html) defined in the Rust
    standard library.
  * Errors can be wrapped into a custom error defined by the library. Without this, it would be
    impossible to differentiate between errors from different libraries. An IO error in `Foo`
    would be indistinguishable from an IO error in `Bar`
  * It increases the cardinality of errors at the application level. High cardinality data allows
    us to answer very specific questions, giving us the insights we need. This helps when you need
    to understand error rates across your system, find areas that need some extra maintenance, debug
    production issues and so on.
* When defining custom errors, they should always implemented the standard library `Error` trait,
  `std::error::Error`. This is to mitigate compatibility issues with errors from other libraries.
* Libraries should never panic. From the application programmer's point of view, panics are undefined
  behaviour; there is no expectation that a library call will crash an application. Errors should be
  bubbled up to the caller.
* Applications consume errors and make informed decisions on how to handle them. There is often not a
  huge need for custom errors at the application level. Additionally, at this level, panicking may be
  the best option and is therefore reasonable.
* Applications are responsible for deciding how errors are formatted and displayed to users.

As we can see, there are subtly different requirements depending on which one you are writing. Rust
being Rust has crates for both these use cases. The most popular of these are [anyhow] and [thiserror].
`anyhow` is used for error reporting in applications while `thiserror` is used for creating custom
errors for libraries (and applications). I will go through their use cases in context of important
aspects of error handling.

### It depends

A while ago, I tweeted this

{{< tweet 1363729866194116608 >}}

As (not) funny as it may be, the truth is that most technical decisions (and arguably most life decisions)
come down to *context*. If you operate outside of context, you're likely to make a whole host of sub-optimal
decisions; not because they are inherently incorrect, they're just not applicable to the situation. Similarly,
if error messages are missing the context in which they are generated, they're likely to send you down
a spiral of debugging. Most of that time might unfortunately be spent exploring dead ends. For errors
to be informative, they need to include additional context. Taking a simple example, imagine you are
trying to open a file and it does not exist. If we printed the error, we would get

```
No such file or directory (os error 2)
```

This is barely useful. We know that we have an IO error but no idea what generated it. We also do not
know which file or directory is the culprit. Now imagine our error message was like this

```
Error: Failed to read file /path/to/directory/ferris.txt

Caused by:
    No such file or directory (os error 2)
```

Much better! We know which file we are missing and where it's located. If we let our imagination run
unbounded, we can think of additional context we could add - like a stack backtrace.

```
Error: Failed to read file /path/to/directory/ferris.txt

Caused by:
    No such file or directory (os error 2)

Stack backtrace:
   0: std::backtrace_rs::backtrace::libunwind::trace
             at /rustc/f5f33ec0e0455eefa72fc5567eb1280a4d5ee206/library/std/src/../../...
      std::backtrace_rs::backtrace::trace_unsynchronized
   ...
   ...
```

As you can see, adding additional context makes our errors so much more informative. This aids us
whenever we need to debug our code.

As I mentioned earlier, I said I'd talk about the relevant crates as we go over various topics. `anyhow`
allows us to add additional context to our error messages. The above two error reports are produced by
`anyhow`.

```Rust
use std::fs;
use anyhow::{self, Context};

fn main() -> anyhow::Result<()>{
    let path = "ferris.txt";
    let content = std::fs::read(path)
        .with_context(|| format!("Failed to read file {}", path))?;

    Ok(())
}
```

### Be like Mansory

[Mansory](https://en.wikipedia.org/wiki/Mansory) is a luxury car modification company. Their cars are
something to behold and I'm not even that into cars. In the same way they make crazy custom cars, we
should strive to make crazy custom errors. Okay, maybe not *too* crazy. Libraries should have their own
set of custom errors that are meaningful given the domain they operate within. In other cases, they should
wrap standard errors. As mentioned, this ensures we can differentiate between similar classes of errors
between libraries. You can also take the approach [tokio] took and [re-export types](https://docs.rs/tokio/1.4.0/tokio/io/index.html#reexports)
so they are accessible through your library but still differentiated from another library. Once again, when
you strip it down to its core essence, the point of all of this is communication. We want our libraries to
faithfully communicate to the developer what type of error was encountered when things go wrong.

Rust requires a bit of ceremony to [define custom types](https://learning-rust.github.io/docs/e7.custom_error_types.html).
It is worth digging into before using a library to do the heavy lifting for you. `thiserror` is a library
used for creating custom errors. I've found it enjoyable to use, albeit I have not had to do anything
advanced. Imagine we have a library `beatmaker` that generates music using midi. The errors we care about:

* The notes are invalid music notes i.e they are not between A & G
* The format of the file containing the notes is invalid
* The instrument is invalid - it's not in the set of instruments we have
* IO errors e.g the file is missing. For IO errors, we want to wrap the IO error into one of our own
  errors.

```Rust
use thiserror::Error

#[derive(Error, Debug)]
enum BeatMakerError{
    #[error("The music notes are not all valid. Please ensure they are between A & G")]
    InvalidNotes,
    #[error("The format of your .bm file is invalid. Check the guide to learn how to create .bm files")]
    InvalidFormat,
    #[error("The name of your instrument is invalid. Please check the instrument list for all valid instruments")]
    InvalidInstrumentName,
    // This wraps all IO errors produced by the std lib into our defined IOError
    #[error(transparent)]
    IOError(#[from] std::io::Error), 
}
```

We can use it in an application like so

```Rust
fn raise() -> Result<(), BeatMakerError> {
    return Err(BeatMakerError::InvalidNotes)
}

fn main() -> Result<(), BeatMakerError> {
    match raise() {
        Ok(_) => println!("All good!"),
        Err(e) => println!("I'd write better errors but I'm lazy. The error: {}", e)
    }

    Ok(())
}
```

Since we've wrapped the standard library's IO errors, we can write a fallible function that returns an
IO error and it will work seamlessly with `BeatMakerError`

```Rust
use std::fs::File;

fn io_raise() -> Result<(), BeatMakerError> {
    File::open("ferris.txt")?;
    Ok(())
}

fn main() -> Result<(), BeatMakerError> {
    match io_raise() {
        Ok(_) => println!("All good!"),
        Err(e) => println!("I'd write better errors but I'm lazy. The error: {}", e)
    }

    Ok(())
}
```

On error, this gives us the error message

```
I'd write better errors but I'm lazy. The error: No such file or directory (os error 2)
```

## The future of errors

[The future of errors]: #the-future-of-errors

One aspect of Rust's community I really enjoy is its grandiose imagination. In particular, Rust's language
designers have a seemingly unbounded imagination. They dream of designing a language that is overwhelmingly
pleasant to use. In many ways, they are getting this right - adoption for the language is growing even
in the face of complexity such as the borrow checker. Error handling is no different. In reading more
about error handling, I came across [Jane Lusby]'s RustConf 2020 talk, [Error handling Isn't All About
Errors]. She wrote [eyre], a fork of `anyhow` that adds support for customised error reports. I encourage
you to watch the video and check out the library. The core focus of `eyre` is to further improve error
messages/reports. It's imagining even more informative errors in Rust.

```
Error:
   0: Unable to read config
   1: cmd exited with non-zero status code

Location:
   src/main.rs:50

Stderr:
   cat: fake_file: No such file or directory

  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ BACKTRACE ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                                ⋮ 10 frames hidden ⋮
  11: <std::process::Command as solo::Output>::output2::h8837a51b9856a548
      at /Users/senyosimpson/Projects/test/solo/src/main.rs:50
  12: solo::read_file::h21ce0bfb8b0da736
      at /Users/senyosimpson/Projects/test/solo/src/main.rs:88
  13: solo::read_config::h048c15951f2a6b11
      at /Users/senyosimpson/Projects/test/solo/src/main.rs:93
  14: solo::main::hec1c621ee896fc4a
      at /Users/senyosimpson/Projects/test/solo/src/main.rs:65
  15: core::ops::function::FnOnce::call_once::h94986c4cb4784c0a
      at /Users/senyosimpson/.rustup/toolchains/nightly-x86_64-apple-darwin/lib/rustlib/..
                                ⋮ 9 frames hidden ⋮

Suggestion: try using a file that exists next time
```

The above error report is an example of this. There are some nice novelties. The error contains both
the error messages and the order in which they occurred in a nice format. There is also a suggestion
which is something we do not see often in standard error reports. The other information we are used
to seeing but it is nicely formatted. It really contains all the information you need in a nice and
succinct format. There's even possibilities of extending this further. If we have a chain of errors,
we can generate a report like this

```
Error:
   0: encountered multiple errors

Location:
   src/main.rs:47

Error:
   0: The task could not be completed
   1: The task you ran encountered an error

Error:
   0: The machine is unreachable
   1: The machine you're connecting to is actively on fire

Error:
   0: The file could not be parsed
   1: The file you're parsing is literally written in c++ instead of rust, what the hell

  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ BACKTRACE ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                                ⋮ 10 frames hidden ⋮
  11: solo::join_errors::hd118537501e8ddf7
      at /Users/senyosimpson/Projects/test/solo/src/main.rs:47
  12: solo::main::hec1c621ee896fc4a
      at /Users/senyosimpson/Projects/test/solo/src/main.rs:35
  13: core::ops::function::FnOnce::call_once::h94986c4cb4784c0a
      at /Users/senyosimpson/.rustup/toolchains/nightly-x86_64-apple-darwin/lib/rustlib/..
                                ⋮ 9 frames hidden ⋮
```

This is so cool! We get all the information we need with the full chain of errors. The error
messages are also stellar, nice touch Jane :)

We have the capacity to imagine a future with the most enjoyable and ergonomic error handling
and informative, easily readable error reports. It's great to see there is active work on this
front. Hopefully, Rust will continue to make our lives all a bit better, one error message at a
time.

***

Shoutouts to [Ana] for reviewing this post 🐻

[Error Handling Project Group]: https://github.com/rust-lang/project-error-handling
[anyhow]: https://docs.rs/anyhow/1.0.39
[thiserror]: https://docs.rs/thiserror/1.0.24/thiserror/
[tokio]: https://docs.rs/tokio/1.4.0/tokio/
[Error handling Isn't All About Errors]: https://www.youtube.com/watch?v=rAF8mLI0naQ
[eyre]: https://docs.rs/eyre/0.6.5/eyre/
[Jane Lusby]: https://twitter.com/yaahc_
[Ana]: https://twitter.com/a_hoverbear
[Pokemon exception handling]: https://wiki.c2.com/?PokemonExceptionHandling
[ML family of languages]: https://en.wikipedia.org/wiki/ML_(programming_language)
