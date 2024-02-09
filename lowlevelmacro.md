# Building a low level macro in rust
## Introduction
Macros in rust can be created in various ways. Declarative macros use the `macro_rules!` macro 
to generate a macro based on pattern matching. The disadvantage of this method that it requires you 
to learn a new regular expression-like language.
Furthermore, declarative macro's have their limitations. You can't build all macros declaratively.

Procedural macros, on the other hand, are build using regular rust code. But even in this case there 
are often a lot of abstractions added to make the writing process quicker and easier. Many procedural 
macros rely on the syn crate, which pre-parses tokens and condenses them into higher level language constructs. 
In addition to that, macros like `quote!` are used to write out the code the macro needs to generate as plain text.

All these abstractions allow you to create most macros quickly, but in each case, you have to learn these tools
and you have to deal with the quirks and limitations that come with these.

Another disadvantage of these methods is that these extra layers of abstraction may slow down the compilation
process as each layer needs it's own build time.

Learning how to write macros at the lowest possible level, gives you all the flexibility to write the most 
demanding macros, while giving you better insight into how rust manages macros under the hood.

## Our tutorial macro
The macro we will be building will allow us to generate an enum and a number of functions based on a 
list of error declarations.\
This is what the macro usage will look like::
```
define_errors!(
    WrongValue : E : "The value {value} is wrong.",
    TooSmall : W : "Warning: the value {value} is smaller than {limit}",
)
```
Note that this macro does not contain valid rust syntax. We will be implementing our own syntax, 
in the same way that an `sql!` macro accepts sql syntax. 
In addition to that, the macro will scan the error message for `{place_holders}` that need to be replaced 
with a format operation. These placeholders will be used as function arguments with the same name.

The resulting code of the above macro statement will be:
```rust
pub enum ErrorId {
    WrongValue,
    TooSmall,
}

pub fn wrong_value(value: &str) -> Error {
    Error {
        id: ErrorId::WrongValue,
        error_type: ErrorType::E,
        message: format!("The value {value} is wrong.", value=value)
    }
}
pub fn too_small(value: &str, limit: &str) -> Error {
    Error {
        id: ErrorId::TooSmall,
        error_type: ErrorType::W,
        message: format!("Warning: the value {value} is smaller than {limit}", value=value, limit=limit)
    }
}
```

## Getting started

To get set up, you can follow the short tutorial in '[the book](https://doc.rust-lang.org/book/ch19-06-macros.html)'.
Scroll down to the section "How to Write a Custom derive Macro".
If you're in a hurry, here's a repo with an empty macro project: TODO

### Project setup
Our project should have a `macro` lib crate, a `macro_derive` lib crate 
[TODO: are both really necessary, if we aren't building a derive macro?]
and a `main` or `test` binary crate to try out our macros.

## TokenStream and TokenTree

In your `macros/macros_derive/src/lib.rs` file, add this macro:

```rust
#[proc_macro]
pub fn print_tokens(input: TokenStream) -> TokenStream {
    println!("TOKENSTREAM::");
    println!("{:?}", input);
    TokenStream::new()
}
```

Optionally, you may want to re-export the macro function in your `macros/src/lib.rs` file:
```rust
pub use macros_derive::print_tokens;
```

If you haven't done so yet, set up a main project to use the macros and let's see what 
the invocation of the macro will print for this small function:
```rust
fn small_function(arg1: &str, arg2: Option<u32>) -> bool {
    true
}
```
TODO: show the output.

Our `small_function` has been converted from plain text to a stream of Tokens, or more specifically, 
a stream of `enum TokenTree`variants. All possible tokens can be represented with this surprisingly short enum:
```rust
//  proc_macro::
pub enum TokenTree {
    Group(Group),
    Ident(Ident),
    Punct(Punct),
    Literal(Literal),
}
```
The reason this is called a `TokenTree` and not just `Token`, is because the `Group` variant nests a deeper
level TokenStream, thus resulting in a tree structure.

## TODO
* Possible exercises: add `#[inline]` to the functions.
* The enum and functions are `pub`. Perhaps make this customizable.