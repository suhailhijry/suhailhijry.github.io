---
title: "Thoughts on Parsing and Compiler Design"
date: 2023-02-27T17:17:10+03:00
draft: false
---

# Background

_I'm omitting many details in this post, since parsing is a very broad topic, and diving into each point will be out of the scope of this post_

A parser is usually that part of a compiler/program that translates some data into other form of data. It can be a CSV, JSON, YAML, or even a whole programming language parser. A parser follows a grammar, and just like any natural language, programming languages, data exchange formats, and even file formats have grammar rules. However, in this post I'll mostly focus on programming language compilers and specifically parsers.

### Compilers

Compilers are usually split into phases, with every phase doing one type of transformation on the program's text. So it's usually split into a scanner or a tokenizer, a parser, a type checker (for statically typed languages), and a code generator.

There might be additional phases to a compiler, but that depends on the language of course. For example, the rust compiler has an additional "borrow checker", where it enforces single ownership of the program's resources at compile time.

```rust

fn main() {
    let mut s = String::from("Hello, ");
    let s2 = &mut s;
    s2.push_str("world!");

    // this will result in a compiler error, since 's' resources are transfered to 's2'.
    // and since 'println!' also wants to borrow 's' in order to print it's content, it
    // cannot borrow any moved resources, and this is validated at compile time during
    // the borrow checker's phase.
    //    println!("s: {}, s2: {}", s, s2);

    // we cannot use our original 's', since its resources (memory) are moved to s2, so
    // use s2 instead.
    println!("s2: {}", s2);
}

```

There are many details regarding the borrow checker and how it works, and a single blog post cannot cover them all, this is just to demonstrate that a compiler can have arbitrary phases to do transformation or checking on the source code.

### "Analysis"

Normally, each phase does only a small transformation on the text, the scanner may convert the characters into tokens which are units representing a higher level view of the text. Each token can be a symbol like ```+``` or a name like ```foo``` or a keyword like ```for``` in C, C++, and Rust.

A scanner would usually read the source code character by character until it can form a meaningful token. For example providing the text ```a + b``` to the scanner would give us the following list of tokens:

```
[name('a'), symbol('+'), name('a')]
```

A parser on the other hand, would take these tokens and transform them into a parse tree or an abstract syntax tree, or both, by reading each token produced by the scanner and creating a more meaningful representation of the text.

For example our previous token list for ```a + b``` would become:

```
                                    binary(+)
                                       /  \
                                      /    \
                                name('a')   name('b')

```

But how exactly we would do something like that?

### Parsing Algorithms and Abstract Syntax Trees

It turns out that efficient algorithms for parsing are a very important topic in computer science and even linguistics. Basically when creating a language we have to make up rules for it, and these rules can either be context-sensitive like English and Arabic (which are natural languages) or context-free like almost all programming languages in existence. From these rules we can make assumptions on how the language might be used and parse it accordingly, discarding any text outside the grammar as an error.

In a context-sensitive grammar this would be very hard, since the meaning of each word depends on the context in which the word was used, which would naturally make natural language parsing an extremely complicated endeavor. That's why almost all programming languages define context-free grammar.

The Backus-Naur Form for [JSON](https://github.com/JetBrains/Grammar-Kit/blob/master/testData/livePreview/Json.bnf) (courtesy of JetBrains):


```bnf

{
  tokens = [
    space='regexp:\s+'
    string = "regexp:\"[^\"]*\"|'[^']*'"
    number = "regexp:(\+|\-)?\p{Digit}*"
    id = "regexp:\p{Alpha}\w*"
    comma = ","
    colon = ":"
    brace1 = "{"
    brace2 = "}"
    brack1 = "["
    brack2 = "]"
  ]
  extends("array|object|json")=value
}

root ::= json
json ::= array | object  { hooks=[wsBinders="null, null"] }
value ::= string | number | json {name="value" hooks=[leftBinder="GREEDY_LEFT_BINDER"]}

array ::= '[' [!']' item (!']' ',' item) *] ']' {pin(".*")=1 extends=json}
private item ::= json {recoverWhile=recover}
object ::= '{' [!'}' prop (!'}' ',' prop) *] '}' {pin(".*")=1 extends=json}
prop ::= [] name ':' value {pin=1 recoverWhile=recover} // remove [] to make NAME mandatory
name ::= id | string {name="name" hooks=[rightBinder="GREEDY_RIGHT_BINDER"]}
private recover ::= !(',' | ']' | '}' | '[' | '{')

```

Defining the grammar using BNF isn't necessary, but BNF, Extended-BNF, or any other form, is helpful when analyzing a language, and ensuring the parsing behavior is predictable.

A parser for JSON would follow the previous grammar, and try to parse the language accordingly, producing the said Abstract Syntax Trees in the process, which then we would transform into whatever representation we like, be it an in memory structure or another file format.

# Complexity

Some languages define grammar that are ambiguous (non-deterministic) in which tokens can be reduced to multiple meanings depending on the context. One popular example of this is C++'s [most vexing parse](https://en.wikipedia.org/wiki/Most_vexing_parse).

C++'s grammar is almost always ambiguous, and depending on the algorithm you use, parsing will be either hard or impossible without hacking the parser to get symbol table data during the AST construction.

### To generate, or not to generate

Many compiler designs either use a hand written parser or a parser generator, in which you provide the generator with the grammar, and it produces a parser for your language.

The hand written parser almost always results in faster compilers, and much better diagnostics for the language, however it is error prone and requires thorough testing, while the generated one will always do right thing provided the grammar is defined properly and the generator itself is correct.

Many proponents to the generation approach easily forget that compilers are essentially a tool, and the user interface part is the most important part of the tool. This means good and verbose diagnostic reports and fast compilation times are a must for any compiler.

### Lazy parsers

From what we learned so far, compilers are essentially multiple transformation tools composed together, each more complex than the last, and many programmers like to draw a very clear line between those tools: a scanner must only do text to token transformation, a parser must do token to syntax tree transformation and so on.

But we forget one important thing: we are essentially problem solvers, and ambiguous grammar are just another problem.

We have tools and algorithms to verify if a particular grammar is ambiguous or not. Not only that, but exactly where the ambiguity lies, so hand written parsers can be just as correct given they handle ambiguity accordingly. But the question is, when do we handle ambiguity?

The parser's main job is to derive meaning from words and symbols. For ambiguous grammar, we could simply write "lazy parsers". parsing the text normally until we reach the possibly ambiguous parts (which we know by using other tools to verify the grammar), the parser simply parses all possible interpretations into a new "ambiguous" node, which holds a list of the possible interpretations of the tokens.

This is essentially what [Elkhound](https://github.com/WeiDUorg/elkhound) does, it generates a parser that defers the ambiguity resolution.

For more info on Elkhound, see [this lecture](https://www.youtube.com/watch?v=uncfFsbUF68).

# Conclusion

Compiler phases are ways to "divide and conqure" the problem of compiling a language, and the parser is only one step towards the end goal, reasonably adding additional steps/phases would only make it easier to write compilers, without having to invent new, complex, and slow algorithms.

