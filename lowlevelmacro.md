# Building a low level macro in rust
{:.no_toc}
## Table on contents
{:.no_toc}
* TOC
{:toc}

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
### Creating a print_token macro
In your `macros/macros_derive/src/lib.rs` file, add this macro:

```rust
#[proc_macro]
pub fn print_tokens(input: TokenStream) -> TokenStream {
    println!("{:?}", input);
    TokenStream::new()
}
```

Optionally, you may want to re-export the macro function in your `macros/src/lib.rs` file:
```rust
pub use macros_derive::print_tokens;
```

If you haven't done so yet, set up a main project to use the macros and let's see what 
the invocation of the macro will print for this small function. In your `main()` function, add:
```rust
print_tokens!(
    fn small_function(arg1: &str, arg2: Option<u32>) -> bool {
        let message = "The message";
        let i = i32::MIN_VALUE;
        true
    }
);
```

### Analyzing the input TokenStream

If you manually reformat the output a bit, you'll get something lke this. This looks like a lot, but if you read line
by line, you'll quickly find out it's quite a straightforward translation of the input. I've also added some comments in some places.
```
TokenStream [
    Ident { ident: "fn", span: #0 bytes(95..97) }, 
    Ident { ident: "small_function", span: #0 bytes(98..112) },
       //[ed.] A group is defined by it's opening delimiter '('. The closing one ')' is implied. 
    Group { delimiter: Parenthesis, stream: TokenStream [ 
        Ident { ident: "arg1", span: #0 bytes(113..117) }, 
        Punct { ch: ':', spacing: Alone, span: #0 bytes(117..118) }, 
        Punct { ch: '&', spacing: Alone, span: #0 bytes(119..120) }, 
        Ident { ident: "str", span: #0 bytes(120..123) }, 
        Punct { ch: ',', spacing: Alone, span: #0 bytes(123..124) }, 
        Ident { ident: "arg2", span: #0 bytes(125..129) }, 
        Punct { ch: ':', spacing: Alone, span: #0 bytes(129..130) }, 
        Ident { ident: "Option", span: #0 bytes(131..137) }, 
          //[ed.] Note how genaric params are not considered a Group.
        Punct { ch: '<', spacing: Alone, span: #0 bytes(137..138) }, 
        Ident { ident: "u32", span: #0 bytes(138..141) }, 
        Punct { ch: '>', spacing: Alone, span: #0 bytes(141..142) }
        ], span: #0 bytes(112..143) 
    }, 
        //[ed.] The double collon are 2 separate Puncts, that are `Joint`.
    Punct { ch: '-', spacing: Joint, span: #0 bytes(144..145) }, 
    Punct { ch: '>', spacing: Alone, span: #0 bytes(145..146) }, 
    Ident { ident: "bool", span: #0 bytes(147..151) }, 
    Group { delimiter: Brace, stream: TokenStream [
        Ident { ident: "let", span: #0 bytes(162..165) }, 
        Ident { ident: "message", span: #0 bytes(166..173) }, 
        Punct { ch: '=', spacing: Alone, span: #0 bytes(174..175) }, 
        Literal { kind: Str, symbol: "The message", suffix: None, span: #0 bytes(176..189) }, 
        Punct { ch: ';', spacing: Alone, span: #0 bytes(189..190) }, 
        Ident { ident: "let", span: #0 bytes(199..202) }, 
        Ident { ident: "i", span: #0 bytes(203..204) }, 
        Punct { ch: '=', spacing: Alone, span: #0 bytes(205..206) }, 
        Ident { ident: "i32", span: #0 bytes(207..210) }, 
        Punct { ch: ':', spacing: Joint, span: #0 bytes(210..211) },
        Punct { ch: ':', spacing: Alone, span: #0 bytes(211..212) }, 
        Ident { ident: "MIN_VALUE", span: #0 bytes(212..221) }, 
        Punct { ch: ';', spacing: Alone, span: #0 bytes(221..222) }, 
        Ident { ident: "true", span: #0 bytes(231..235) }
        ], span: #0 bytes(152..241) 
    }
]
```

Our `small_function` has been converted from plain text to a stream of Tokens, or more specifically, 
a stream of `enum TokenTree`variants. All possible tokens can be represented with this surprisingly short enum:
```rust
//  proc_macro:
pub enum TokenTree {
    Group(Group),
    Ident(Ident),
    Punct(Punct),
    Literal(Literal),
}
```
The reason this enum is called a `TokenTree` and not just `Token`, is because the `Group` variant nests a deeper
level TokenStream, thus resulting in a tree structure.

* `Ident`: All alphabetic 'words' are represented as `Ident`, including keywords lke `fn` or type names like `u32`.
* `Punct`: Special characters are represented as `Punct`. This includes `&`, commas, dots,... Note that `::` is represented by 2 separate punctuations!
* `Literal`: In our case we have 2 literals: a string and a number.
* `Group`: Everything that is placed between braces, brackets and parenthesis, are placed in a `Group`.

If you haven't written any macros yet, take a look at the signature of the `print_tokens()` macro function:
```rust
fn print_tokens(input: TokenStream) -> TokenStream {}
```
It takes a `TokenStream` as input and returns an output `TokenStream`. That's it. This means that if we understand 
how the rust compiler creates `TokenStream`s, we will be able to create our own. So, take some time to analyze
the above output.

## Building the enum

Our goal is to create a macro that generates an enum and functions from error definitions.
Let's start with a simplified macro that will just create an enum.
```
define_errors!(
    WrongValue,
    TooSmall,
)
```
In this version, we simply have to wrap the content of the macro call with
```rust
pub enum ErrorId {
    //content of macro.
}
```
In tokens, that would be 3 `Ident`s and ` `Group`.

## Parsing the error definitions
todo
## Handling parsing errors.
todo
## Building the functions.
todo

## Improvements
Here are some improvements you could make to the macro we built:
* Possible exercises: add `#[inline]` to the functions.
* The enum and functions are `pub`. Perhaps make this customizable.
* The name of the enum is currently hard coded. Make this user-defined.