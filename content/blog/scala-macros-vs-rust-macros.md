+++
title = "Scala Macros vs. Rust Macros"
date = "2016-09-04"
type = "post"
tags = ["Scala", "Rust", "Macros", "Metaprogramming"]
+++

Having worked with Scala for some years now, I have used Scala macros on several occasions and always have been impressed by how powerful they are.
Recently, I started learning Rust and also came across its macro system.
Although both metaprogramming facilities might seem similar at first sight, the way they work is actually quite different.
In the following I will explore briefly how Rust macro rules and Scala macros work, how they are different and compare them against each other.

## Scala Macros

[Scala macros](http://scalamacros.org/paperstalks/2013-04-22-LetOurPowersCombine.pdf) can be seen as metaprograms that have knowledge of the program structure and the ability to alter it.
Macros are recognized by the compiler as methods to be invoked during compilation.
The compiler will expand the macro implementation which results in an abstract syntax tree (AST) representing a Scala program or a part of it.
The generated AST is then inlined at the caller site where the macro has been invoked.
In this way, the final code contains no reference to the macro because the macro invocation is replaced by the resulting AST.

Within macros, the representation of the program to be compiled is exposed.
Scala offers an API that provides routines to manipulate the exposed program code as well as other methods for parsing, type checking, and error reporting.
Generally, Scala macros work on the _abstract syntax tree_.

As can be seen in the following code example, Scala macros consist of two parts: the _macro definition_ and the _macro implementation_.


```Scala
import scala.reflect.macros.blackbox.Context
import scala.language.experimental.macros

// macro definition
def printFields[T](obj: T): String = macro printFieldsImpl[T]

// macro implementation
def printFieldsImpl[T: c.WeakTypeTag](c: Context)(obj: c.Expr[T]): c.Tree = {
  import c.universe._

  // get all fields of obj
  val fields: Iterable[c.Symbol] = obj.tree.tpe.decls
    .filter(sym => sym.isMethod && sym.asTerm.isParamAccessor)

  // return field identifiers
  val x = q" ..${ fields map { field => field.name.toString } } "  
  q""" "Fields: " + $x """
}

// case class User(name: String, age: Int)
// printFields(User("John Doe", 33)) => "Fields: (name,age)"
```

Macro definition, like `printFields` look almost like normal function definitions in Scala.
However, the function body starts with the keyword `macro` and is followed by the identifier of the macro implementation.

Macro implementations, such as `printFieldsImpl`, contain the actual logic executed during compile time.
The parameters as well as the return type of a macro implementation are expression trees, which wrap the AST.
There are several ways to construct an expression tree within a macro implementation.
One of the most convenient ways is to use [quasiquotes](http://docs.scala-lang.org/overviews/quasiquotes/intro.html).
Quasiquote expressions are snippets of the following form: `q"…[expressions]…"`, where `expressions` can be a snippet of Scala code that will be transformed into an expression tree.

In the example above, the first expression in the macro iterates over all fields of the parameter type passed to the macro and creates a collection of the field names.
The second expression concatenates the string `"Fields: "` to the result of the first expression.
Finally, an expression tree representing this created string will be returned.

Another thing worth noting is that the macro in our example is generic.
Instead of forcing to have parameters with a specific type, this macro can be used for function parameters with any type.

Currently, macros are still an experimental feature in Scala.
Therefore, to use them, the package `scala.language.experimental.macros` has to be imported.
As macros influence the compilation process, they cannot be defined and used in the same compilation unit from where they are invoked.
Instead, the project has to be split in at least two separated sub-projects with one of them containing the macro definitions.


## Rust Macros

In Rust, macros are used for syntax extension.
Just like in Scala, they are expanded during compile time even before any static checking.
The AST is traversed and all macro invocations are replaced with their expansion.

The following example shows the definition of a simple Rust macro that logs a message with a specific timestamp:

```rust
extern crate chrono;

macro_rules! log {
  ($msg:expr) => (println!("Log {}: {}", chrono::Local::now(), $msg));
}

//log!("Error - Division by zero");
//→ Log 2016-09-03 17:39:18.773956 +02:00: Error - Division by zero
```

Macro definitions start with the keyword `macro_rules!` followed by the macro's name.
The body of the macro definition consists of rules expressed as pattern-matching cases.
On the left, these pattern-matching cases describe the part of the AST that should be matched.
Rust offers a [specific grammar](https://doc.rust-lang.org/reference.html#macros) for describing how what parts of the AST should be matched.
The right part of these cases consists of normal Rust code.
It is also possible to capture values, such as in the example above, by using variables that are preceded by `$` and then added right before the AST part that should be captured.
To use defined macros, a bang (`!`) needs to be added at the end of the macro name in order to distinguish them from normal function calls.

Unlike macros in Scala, macros in Rust work on _token trees_.
The inputs of a macro are token trees which are not interpreted by the parser.
These token trees are based on lexical information and therefore macros do not have any type information.

Another great quality of Rust's macro system is _hygiene_. 
This means that no shadowing of variables with the same name can occur because each macro expansion happens in a distinct _syntax context_.
As each variable is tagged with the specific syntax context where it was introduced, there will be no conflicts.
Although there are several workaround to achieve hygiene in Scala and with quasiquotes, currently [automatic hygiene is not supported](http://docs.scala-lang.org/overviews/quasiquotes/hygiene).


## Comparison and Conclusion

Now that we have a basic idea about how macros in both languages work, the following table gives a short summary and directly compares some of their features.
I have also included some information about debugging macros and additional aspects.


&nbsp;  |  Scala Macros  | Rust Macros
-----|  ------------- | -------------
Macro Input | Expression Trees with type information. | Token Trees with lexical information.
Macro Return Type | Expression Tree. | No declared return type.
Macro Declaration | A macro consist of two parts: the _macro definition_ containing the keyword `macro` and the _macro implementation_. | Macros are declared using `macro_rules! <name> {...}`
Implementing Macros | Expression trees within macros can be constructed by writing the AST manually using [tree classes](http://www.scala-lang.org/api/2.9.2/scala/reflect/generic/Trees.html), by employing the `reify` macro or by utilizing [quasiquotes](http://docs.scala-lang.org/overviews/quasiquotes/intro.html). | Rules are defined using pattern-matching cases that match specific parts of an AST and specify the Rust code to be executed.
Hygiene | Automatic hygiene is currently (Scala 2.11) not supported, but will probably added in [the future](https://issues.scala-lang.org/browse/SI-7823). | Rust macros are hygienic. 
Recursion | Scala macros can call themselves. | Macro implementations can contain invocations of themselves. Macros have a specific recursion limit that can be increased by adding `#![recursion_limit="…"]`
Debugging | The `-Ymacro-debug-lite` option will output generated code and the raw AST representation after the macro expansion. There is also `showRaw(...)` which can be used within the source code and will output the raw AST of an expression tree provided as parameter. | To view the expanded macro: `rustc --pretty expanded`. To show syntax context information: `rustc --pretty expanded,hygiene`. To show a compiler messages every time a macro expands add `trace_macros!(true)`. `log_syntax!(...)` outputs provided parameters at compile time.
Project Setup | Separated sub-projects for macro definitions. | Can be in the same source file in which they are used. Can be directly used after their definition. To use them in other files or projects they need to be [exported/imported](https://doc.rust-lang.org/book/macros.html#scoping-and-macro-importexport).
Areas of Application | code generation, program self-optimization, DSLs, code inspection, static type checking  | syntax extension
More Information| [Scala Documentation about Macros](http://docs.scala-lang.org/overviews/macros/overview.html)  | [Rust Macros in "The Book"](https://doc.rust-lang.org/book/macros.html)


In both languages, macros are expanded early during compile time and provide means for compile-time metaprogramming.
However, the way they work and the features they provide are quite different.
All in all, for me Scala macros feel much more powerful compared to macro rules in Rust, as they are able to inspect types and are not solely based on lexical information. 

For more advanced compile-time metaprogramming Rust offers the possibility to create [compiler plugins](https://doc.rust-lang.org/book/compiler-plugins.html). 
These are more powerful but also more complex to define.

