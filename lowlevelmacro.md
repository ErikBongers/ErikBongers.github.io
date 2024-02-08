# Building a low level macro in rust
## Introduction
Macro's in rust can be created in various ways. Declarative macros use the `macro_rules!` macro 
to generate a macro based on pattern matching. The disadvantage of this method that it requires you 
to learn a new regular expression-like language.
Furthermore, declarative macro's have their limitations. You can't build all macros declaratively.

Procedural macros, on the other hand, are build using regular rust code. But even in this case there 
are often a lot of abstractions added to make the writing process quicker and easier. Many procedural 
macros rely on the syn crate, which pre-parses tokens into higher level language constructs. In addition 
to that, macros like `quote!` are used to write out the code the macro needs to generate as plain text.

All these abstractions allow you to create most macros quickly, but in each case, you have to learn these tools
and you have to deal with the quirks and limitations that come with these.

Another disadvantage of these methods is that these extra layers of abstraction may slow down the compilation
process as each layer needs it's own build time.

Learning how to write macros at the lowest possible level, gives you all the flexibility to write the most 
demanding macros, while helping you to understand how rust manages macros under the hood (almost).

## Our tutorial macro
The macro we will be building will allow us to generate an enum and a number of functions based on a 
list of error declarations.\
This is what the macro usage will look like::
```
define_errors!(
    WrongValue : E : "The value {value} is wrong.",
    TooSmall : W : "Warning: the value {value} is smaller than {limit},
}
```
Note that this macro does not contain valid rust syntax. We will be implementing our own syntax, 
in the same way that an `sql!` macro accepts sql syntax. 