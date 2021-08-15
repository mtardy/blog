---
title: "Summer Internship: Reflexive Programming Language Framework"
description: "This write-up is the extended abstract of my 2020 summer
internship. It was a reflexivity-based programming language project written in
C."
date: "2020-11-30"
tags: ["C", "reflexivity"]
categories: ["programming-language"]
ShowToc: true
---

*This internship was a collaboration between INP-ENSEEIHT, Toulouse, France and
Kyoto University of Advanced Science, Kyoto, Japan. It took place during the
summer of 2020, when the Covid-19 pandemic prevented me from going to Japan, so
it was unfortunately a remote internship. You can find all the sources of the
project in the [github repository](https://github.com/mtardy/sandbox).*

## Abstract

During this summer internship, Pr. Ian Piumarta and myself implemented a
prototype-based programming language, simple and reflexive by design. Programs
written in this language can inspect themselves because the produced abstract
syntax tree is stored in a common data structure. They can also extend the
language functionalities and modify the primitive structures using the language
syntax.

## Introduction

> Programs that are reflexive, i.e., able to inspect and modify their own
implementation, can be extraordinarily flexible. Reflexivity is often provided
by the programming language.

These are the first words of the internship project proposal, written by Pr.
Ian Piumarta `[1]`. During this internship, we implemented a prototype-based
language, simple and reflexive by design. The language proposes a Map object
data structure that is also used to store the Abstract Syntax Tree (AST)
resulting from programs. Thus the program can easily modify itself and evaluate
a program under the form of an AST at run time.

The language syntax and data structures were mainly influenced by C and
JavaScript.[^1]

## Related work

Lisp was originally designed as a practical mathematical notation for computer
programs. It was developed by John McCarthy in 1958 at the MIT [@wiki:lisp]. It
quickly became the favorite language for artificial intelligence research.

> Lisp is the canonical example of such a language where reflexivity is
possible because programs are represented as "first-class" data structures, and
any data can be evaluated as a program dynamically at run time `[3]`. More
recent language design favours different syntax for data and programs, with
programs typically being represented (after parsing) as an abstract syntax tree
(AST). `[1]`

## Details

The starting point of the project was *peg*,[^2] a recursive-descent parser
generator for C, a previous project of Pr. Piumarta. This tool generates a
parser from a Parsing Expression Grammar (PEG) `[4]`. To familiarise myself
with *peg*, I started by writing an elementary grammar, that can parse and
compute simple mathematical expressions: a calculator. The result was similar
to what you can find in peg manual page,[^3] section *LEG EXAMPLE: A DESK
CALCULATOR*. Then we added features, one by one, until it started to look like
a real programming language. Nevertheless, the evaluation of the expressions
were done during parsing, and it started to be too complicated.

To continue improving, we split parsing and evaluation, thus producing an AST
at the end of the parsing. The AST containing all the program information is
then passed to an `eval` function, which is the interpreter of our language,
written in C. We also thought about writing a compiler that would produce
byte-code for an elementary virtual machine. It would have simplified the
management of the execution flow, which was quite complicated to handle with C
`setjmp` and `longjmp`.

Since the beginning, the idea was to produce an AST in the data structure of
the language itself, that would be easy to inspect, so we used our primitive
Map type, which is a key/values pairs store, i.e., a dictionary. Each AST node
contains at least:

-   `prototype`: the parent map of the object, for an AST node, it is a map
    with the name set to the semantic of the node. For example, `Declaration`,
    `Binary`, `While`, `Equal`, `Return`, etc.

-   `line` and `file`: the line and the file of the node definition to produce
    nice error messages in case of runtime errors.

Then other symbols are used and are specific to each node. For example, for the
`Binary` node, there are two additional keys, `lhs` and `rhs`, for the `While`
node, there are `condition` and `body`, etc. This way we had a complete API to
describe any statement of the language in the primitive Map data structure.

As described in the last paragraph, each AST node contains a *prototype*, and
this construction is at the foundation of the programming language we designed
`[5]`. It provides a very flexible way to create hierarchy and inheritance
between objects. The relation between each AST node can even be rewritten in a
bootstrap program if this is useful. For example, you can easily write a
classes mechanism with prototype chaining to implement shared behaviour, just
like in object-oriented programming.

We added many nice features to the language, here is an non-exhaustive and
unordered list:

-   *JavaScript style object access* with `object.property` and `object[exp]`.

-   *Python list slice* with the `array[n:m]`, `array[:n]`, `array[-n]`, etc.

-   *C assignement operator* `+=`, `/=`, etc.

-   *String operations* for concatenation with `+` or multiplication `*` like
    in Perl for example.

-   *Splicing operator* to explode an array into its members.

-   *Runtime error backtrace* for error position and *Throw/Try/Catch mechanism*
    for error execution flow.

-   *Module importation* with `require` keyword like in JavaScript.

-   *Primitive* utility functions, for example, the classical `print`, `keys`
    to get all keys of a map object or `microseconds` to measure time.

-   *Factory functions* to build and convert primitive types.

-   *Syntax* keyword to define functions/macros with no evaluation of the
    arguments.

## Examples

To show a quick glimpse of the language style and syntax, here are three
examples.

### Syntax

Here is a simple example of the `syntax` feature used to extend the language
with a non-evaluated arguments function. On the first line, between parenthesis
are the arguments of the syntax, following is the keyword for the body. The
backtick `(stmt)` is here to declare a statement in which there can be AST
nodes to evaluate, signaled by `@`.

```
syntax until (c) b {
    return `(while (!@c) @b)
}

var x = 0; // var is optional
until (x==10) {
    println(x++)
}
// will print 0 to 9
```

Here is another example, a bit more complexe, of what can be achieved with
`syntax`.

```
syntax foreach (variable, map) block {
    `({
        var _map= values(@map);
        var _len= length(_map);
        for (var _idx= 0;  _idx < _len;  ++_idx) {
            var @(variable.key) = _map[_idx];
            @block;
        }
    });
}
m = { zero: 0, one: 1, eight: "viii" };
foreach ( val, m ) {
    print("v: ", val, "\n");
}
// will print "v: viii", "v: 1" then "v: 0"
```

`values()` and `length()` are primitives of the language. We witness here that
map are unordered collection, in fact they are ordered by keys alphabetical
order to speed up search and insertion.

### Prototype chain

Here is an example of prototype chaining, this is similar of what you would do
in JavaScript to create a common behaviour between two objects.  What is
interesting is that you can implement your own \"new\" constructor mechanism to
create objects.

```
var Object = { __name__: #"Object" };
Object.init = fun () { this; }
Object.new = fun () {
    var obj= { __proto__ : this };
    var init = this.init;
    init && invoke(obj, init, __arguments__);
    obj; // return can be omitted
};       // "{e1; e2;} == e2" -> true

var Point = {
    __name__: #"Point", // #Point is valid too
    __proto__ : Object
};
Point.init = fun (x, y) {
    this.x = x;
    this.y = y;
}
var p = Point.new(3, 4);
Point.foreach = fun (f) {
    f(this.x);
    f(this.y);
}
p.foreach(fun (x) { println(x); })
// will print 3 then 4
```

## Acknowledgments

Thanks to the ENSEEIHT-KUAS collaboration, Pr. Géraldine Morin, and Pr.  Osamu
Tabata that made this internship possible.

Huge thanks to Pr Ian Piumarta who gave me so much of his time during this
summer. That was a real pleasure to work and learn from someone with his level
of experience. Even with the situation that prevents me from going to Japan,
and thus the 7 hours shift, he made easy to exchange and talk. He really knew
how to make me approach a complex project, step by step, and gave me precious
advice. I learned and adopted some of his coding habits and he introduced me to
the insanely efficient C preprocessing programming.

## Conclusion

This internship was really exciting because it's always impressive to see how
we can execute sophisticated code with a relatively small programming language.

I also discovered how designing a programming language is more about discussion
and decision than actually implementing. It makes you think about your
influences and knowledge of other programming languages.  Nevertheless, it was
a great opportunity to improve my C skills anyway and my confidence with the
language.

You can find the project repository on Github at
<https://github.com/mtardy/sandbox> with all the source code, tests files, open
issues and improvements to be addressed if you have some spare time!

## Bibliography
[1]: I. K. Piumarta, Kuas faculty of engineering "internship project proposals
(software engineering) — reflexive programming language framework", 2019.

[2] Wikipedia  contributors, "Lisp(programming  language)", 2020. Available:
https://en.wikipedia.org/wiki/Lisp_(programming_language).

[3] J. McCarthy, "A micro-manual for lisp - not the whole truth", vol. 13, no.
8, pp. 215–216, Aug.  1978, issn: 0362-1340. doi: 10.1145/960118.808386.
Available: https://doi.org/10.1145/960118.808386.

[4] B. Ford, "Parsing  expression  grammars:A recognition-based syntactic
foundation", vol. 39, no. 1, pp. 111–122, Jan. 2004, issn: 0362-1340.
doi: 10.1145/982962.964011. Available: https://doi.org/10.1145/982962.964011.

[5] H. Lieberman, "Using prototypical objects to implement shared behavior in
object-oriented systems,", vol. 21, no. 11, pp. 214–223, Jun. 1986,
issn: 0362-1340. doi: 10.1145/960112.28718. Available:
https://doi.org/10.1145/960112.28718

[^1]: Which is already heavily influenced by C; one can exaggerate that it is
  Scheme with a C syntax

[^2]: Project homepage: <https://www.piumarta.com/software/peg/>

[^3]: Manual page: <https://www.piumarta.com/software/peg/peg.1.html>
