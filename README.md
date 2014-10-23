A long Factor tutorial
======================

Factor is a mature, dynamically typed language based on the concatenative paradigm. Getting started with Factor can be rather daunting, as it follows a paradigm quite far from most mainstream languages. This tutorial will guide you through the basics so that you will be able to appreciate its simplicity and power. I will assume that you are familiar with some functional language, as I will mention freely concepts like folding, higher-order functions or currying.

Even if Factor is a rather niche language, it is mature and features a comprehensive standard library covering tasks from JSON serialization to socket programming or HTML templating. It runs in its own optimized VM, usually reaching top performance for a dinamically typed language. It also has a flexible object system, a FFI with C, and asynchronous I/O - much like node, but with a much simpler model for cooperative multithreading.

In this tutorial, we assume that you have downloaded a copy of Factor and that you are following along with the examples in the listener. The first section gives some motivation for the rather peculiar model of computation, but feel free to skip it if you want to get your feets wet and return to it after some practice.

Concatenative languages
-----------------------

Factor is a *concatenative* programming language in the spirit of Forth. What does this even mean? In a concatenative language, there is essentially only one operation, which is function composition. Since function composition is so pervasive, it is usually implicit, and functions can be literally juxtaposed in order to compose them. So if `f` and `g` are two functions, their composition is just `f g` (usually, functions are read from left to right, so this means first execute `f`, then `g`, unlike in mathematical notation).

This requires some explanation, since functions will usually have multiple inputs and outputs, and it is not always the case that the output of `f` matches the input of `g`. For instance, `g` may need access to values computed by earlier functions. For this to work, `f` and `g` have essentially to be functions which take and return the whole state of the world.

There are various ways this global state can be encoded. The most naive would use a hashmap mapping variable names to their values. This turns out to be too flexible: if every function can access randomly any piece of global state, there is little control on what functions can do, little encapsulation and ultimately programs become an unstructured mess of routines mutating global variables.

It works well in practice to represent the state of the world as a stack. Functions can only refer to the topmost element of the stack, so that elements below it are effectively out of scope. If a few primitives are given to manipulate a few elements on the stack (for instance `swap`, that exchanges the two elements on top), then it becomes possible to refer to values more down the stack, but the farthest the position, the hardest it becomes to refer to it.

So, functions are encouraged to stay small and only refer to the top two or three elements. In a sense, there is no more a distinction between local and global variables, but values can be more or less local depending on their distance from the top of the stack.

Notice that if every function takes the state of the whole world and returns the next state, its input is never used anymore. So, even if it is more convenient to think of pure functions from stacks to stacks, the semantics of the language can be implemented more efficiently by mutating a fixed stack.

This leaves Factor in a strange position, whereby it is both extremely functional - only allowing to compose simpler functions into more complex ones - and largely imperative - describing operations on a mutable stack.

Playing with the stack
----------------------

Let us start looking what Factor actually feels like. Our first words will be literals, like `3`, `12.58` or `"Chuck Norris"`. Literals can be thought as functions that push themselves on the stack. Try writing `5` in the listener and then press enter to confirm. You will see that the stack, initially empty, now looks like

    5

You can enter more that one number, separated by spaces, like `7 3 1`, and get

    5
    7
    3
    1

(the interface shows the top of the stack on the bottom). What about operations? If you write `+`, you will run the `+` function, which pops the two topmost elements and pushes their sum, leaving us with

    5
    7
    4

You can put, as before, more inputs in a single line, so for instance `- *` will leave the single number `15` on the stack (do you see why?). The function `.` (a dot) prints this item, while popping it out of the stack, leaving the stack empty.

If we write everything on one line, our program so far looks like

    5 7 3 1 + - * .

which shows the peculiar way of doing arithmetics by putting the arguments first and the operator last - a convention which is called Reverse Polish Notation (RPN). Notice that this requires no parenthesis, unlike the Lisp convention where the operator comes first, and no precedence rules, unlike most other systems. For instance in any Lisp, the same computation would be written like

    (* 5 (- 7 (+ 3 1)))

Also notice that we have been able to split our computation rather arbitrarily and that each subpiece of our line made sense in itself.

Defining our first word
-----------------------

We will now define our first function. Factor has a slightly odd naming: since functions are just written from left to right, they are simply called **words**, and this is what we will do from now on. Modules in Factor define words in terms of previous words and are then called **vocabularies**.

We will want to compute the factorial. To start with a concrete example, we compute the factorial of `10`, so we start by writing `10` on the stack. Now, the factorial is the product of the numbers from `1` to `10`, so we should produce such a list of numbers first.

The function to produce a range is reasonably called `[a,b]` (tokenization is trivial in Factor, as words are always space separated, and this allows you to use any combination of non-whitespace characters as an identifier). In our case one of the extremes is just `1`, so we can use the simpler word `[1,b]` instead. If you write that in the listener, you will be prompted with a choice, because the name `[1,b]` is not imported by default. Factor is able to suggest to import the `math.ranges` vocabulary, so choose that option and proceed.

You should now have on your stack a rather opaque structure which looks like

    T{ range f 1 10 1 }

This is because our range functions are lazy. To confirm that we actually have created the list of numbers from `1` to `10`, we convert it into an array using the word `>array`. Your stack should now look like

    { 1 2 3 4 5 6 7 8 9 10 }

which is promising.

Next, we want to take the product of those numbers. In many languages, this could be done with a function called reduce or fold. Let us look for one. Pressing `F1` will open a contextual help, where you can search for `reduce`. It turns out that `reduce` is actually the word we are looking for, but at this point it may not be obvious how to use it.

Try writing `1 [ * ] reduce` and look at the output: it is indeed the factorial of `10`. Now, `reduce` usually takes three arguments: a sequence (and we had one), a starting point (and this is the `1`) and a binary operation. This must certainly be the `*`, but what about those square brackets around it?

If we had written just `*`, Factor would have tried to apply multiplication to the topmost two elements, which is not what we wanted. What we need is a way to mention a word without applying it. Keeping our textual metaphor, this mechanism is called **quotation**. To quote one or more words, you just surround them by `[` and `]` (leaving spaces). What you get is akin to an anonymous function in other languages.

Let us `drop` the result to empty the stack, and try writing what we have done so far in a single shot: `10 [1,b] 1 [ * ] reduce`. This will output `3628800` as expected.

We now want to define a word that can be called whenever we want. We will call our word `!` as it is customary. To define it, we need to use the word `:`. Then we put the name of the word being defined, the **stack effects** and finally the body, ending with the `;` word:

    : ! ( n -- n! ) [1,b] 1 [ * ] reduce ;

What are the stack effects? These are the part like `( n -- n! )` in our case. They have no effect on the function, but allow you to name your inputs and outputs for documenting purposes. You can use any identifier to name them, and Factor will only make a consistency check that the number of inputs and outputs agrees with what the body does.

If you try to write

    : ! ( m n -- n! ) [1,b] 1 [ * ] reduce ;

Factor will signal an error that the number of inputs is not consistent. To restore the previous correct definition press `Ctrl+P` two times to get back to the previous input and the enter it.

We can think at the stack effects in definitions both as a documenting tool and as a simple type system, which nevertheless does catch a few errors.

In any case, you have succesfully defined your first word: if you write `10 !` in the listener you can prove it.

Notice that the `1 [ * ] reduce` part of the definition sort of makes sense on its own, being the product of a sequence. The nice thing about a concatenative language is that we can just factor this part out and write

    : prod ( {x1,...,xn} -- x1*...*xn ) 1 [ * ] reduce ;
    : ! ( n -- n! ) [1,b] prod ;

Our definitions have become simpler and there was no need to pass parameters, rename local variables or anything that would have been necessary to factor out a part of the definition in a different language.

Parsing words
-------------

If you have paid attention until now, you will realized that I have lied to you. I have said that each word acts on the stack in order, but there a few words like `[`, `]`, `:` and `;` that certainly seems not to abide to this rule.

In fact these are **parsing words** and behave differently from simpler words like `5`, `[1,b]` or `drop`. We will get into more detail when we talk about metaprogramming, but for now it is enough to know that parsing words are special.

They are not defined using `:`, but using `SYNTAX:` instead. When a parsing words is encountered, it can interact with the parser using a well-defined API to influence how successive words are parsed. For instance `:` asks the next tokens from the parsers until `;` is found and tried to compile that stream of tokens into a word definition.

A common use of parsing words is to define literals. For instance, `{` is a parsing word that starts an array definition and is terminated by `}`. Everything in-between is part of the array. An example of array that we have seen before is `{ 1 2 3 4 5 6 7 8 9 10 }`.

There are also literals for hashmaps, like `H{ { "Perl" "Larry Wall" } { "Factor" "Slava Pestov" } { "Scala" "Martin Odersky" } }`, or byte arrays, like `B{ 1 14 18 23 }`.

Other uses of parsing word include the module system, the object oriented features of Factor, enums, memoized functions, privacy modifiers and more. In theory, even `SYNTAX:` can be defined in terms of itself, although of course the system has to be bootstrapped somehow.

Stack shuffling
---------------

Now that you know the basics of Factor, you may want to start assembling more complex words. This sometimes may require to use variables that are not on top of the stack, or to use variables more that once. There are a few words that can be used to this effect. I will mention them now, since you'd better be aware of them, but warn you that code using many of those words can quickly become hard to write and harder to read. It requires mentally simulating moving values on a stack, which is not a natural way to program. We will see a much more effective way to handle most needs in next section.

I will just write the most common shuffling words together with their effect on the stack. Feel free to try them in the listener, and explore the online help to find out more.

    dup ( x -- x x )
    drop ( x -- )
    swap ( x y -- y x )
    over ( x y -- x y x )
    dupd ( x y -- x x y )
    swapd ( x y z -- y x z )
    nip ( x y -- y )
    rot ( x y z -- y z x )
    -rot ( x y z -- z x y )
    2dup ( x y -- x y x y )

Combinators
-----------

Although the words mentioned in the previous paragraph are occasionally useful (especially `dup`, `drop` and `swap`), one should aim to write code that does as little stack shuffling as possible. This requires a certain practice in putting the function arguments in the right order from the start.

Nevertheless, there are certain patterns of use that are better abstracted away into their own words. For instance, say we want to define a word to determine whether a given number `n` is prime. A simple algorithm would be to test each number from `2` to the square root of `n` and see whether it is a divisor of `n`.

This immediately shows that `n` is used in two places: as an upper bound for the sequence, and as the number to test for divisibility. The word `bi` applies two different quotations to an element on the stack, and this is precisely what we need. For instance `5 [ 2 * ] [ 3 + ] bi` yields

    10
    8

To continue with our example, will need a word to test divisibility, and a quick search in the online help shows that `divisor?` is what we want. We will also need a way to make a range starting from `2`, which we can define like

    : [2,b] ( n -- {2,...,n} ) 2 swap [a,b] ; inline

What's up with that `inline` word? This is one of the modifiers we can use after defining a word, another one being `recursive`. This will allow us to have the definition of the word inlined wherever it is used, rather than incurring a function call.

It may also help to have the arguments for divisibility in the other direction, so we define

    : multiple? ( a b -- ? ) swap divisor? ; inline

Now, producing the range of numbers from `2` to the square root of `n` is easy: `sqrt floor [2,b]` (`floor` is not even necessary, as `[a,b]` works for non-integer bounds). What about the second function?

We must produce a function that test for being a divisor of `n` - in other words we need to partially apply the word `multiple?`. This can be done with the word `curry`, like this: `[ multiple? ] curry`.

Finally, once we have the range and the test function on the stack, we can test whether any element satisfies the divisibility with `any?`. Our full definition looks like

    : prime? ( n -- ? ) [ sqrt [2,b] ] [ [ multiple? ] curry ] bi any? not ;

Altough the definition is slightly complicated, the stack shuffling is minimal and limited to the small helper functions, which are much simpler to reason about than `prime?`.

Notice that our `prime?` word uses two levels of quotation nesting. In general, Factor words tend to be rather shallow, using one level of nesting for each higher-order function, unlike Lisps or more generally languages based on the lambda calculus, which use one level of nesting for each function, higher-order or not.

Many more combinators exists other than `bi` (and its relative `tri`), and you should become acquainted at least with `bi*` and `bi@`.

Vocabularies
------------

It is now time to start writing your functions in files and learn how to import them in the listener. Factor organizes words into nested namespaces called **vocabularies**. You can import all names from a vocabulary with the word `USE:`. In fact, you may have seen something like

    USE: math.ranges

when you asked the listener to import the word `[1,b]` for you. You can also use more than one vocabulary at a time with the word `USING:`, which is followed by a list of vocabularies and terminated by `;`, like

    USING: math.ranges sequences.deep ;

Finally, you define the vocabulary where your definitions are stored with the word `IN:`. If you search the online help for a word you have defined so far, like `prime?`, you will see that your definitions have been grouped under the default `scratchpad` vocabulary. By the way, this shows that the online help automatically collects information about your own words, which is a very useful feature.

There are a few more words, like `QUALIFIED:`, `FROM:`, `EXCLUDE:` and `RENAME:`, that allow more fine-grained control over the imports, but `USING:` is the most common.

On disk, vocabularies are stored under a few root directories, much like with the classpath in JVM languages. By default, the system starts looking up into the directories `basis`, `core`, `extra`, `work` under the Factor home. You can add more, both at runtime with the word `add-vocab-root`, and by creating a configuration file `.factor-rc`, but for now we will store our vocabularies under the `work` directory, which is reserved for the user.

Generate a template for a vocabulary writing

    USE: tools.scaffold
    "github.tutorial" scaffold-work

You will find a file `work/github/tutorial/tutorial.factor` containing an empty vocabulary. You can add the definitions of the previous paragraph, so that it looks like

    ! Copyright (C) 2014 Andrea Ferretti.
    ! See http://factorcode.org/license.txt for BSD license.
    USING: ;
    IN: github.tutorial

    : [2,b] ( n -- {2,...,n} ) 2 swap [a,b] ; inline

    : multiple? ( a b -- ? ) swap divisor? ; inline

    : prime? ( n -- ? ) [ sqrt [2,b] ] [ [ multiple? ] curry ] bi any? not ;

Since the vocabulary was already loaded when you scaffolded it, we need a way to refresh it from disk. You can do this with `"github.tutorial" refresh`. There is also a `refresh-all` word, with a shortcut `F2`.

You will be prompted a few times to use vocabularies, since your `USING:` statement is empty. After having accepted all of them, Factor suggests you a new header with all the needed imports:

    USING: kernel math.functions math.ranges sequences ;
    IN: github.tutorial

You now want to come back and edit the file to add this header. Of course, you still have the file open your editor, but if this was not the case, you could make use of Factor editor integration.

For instance, I am using Sublime Text right now, so I load the integration with

    USE: editors.sublime

After having done that, I can edit, say, the `multiple?` word with `\ multiple? edit`. I find Sublime Text open on the relevant line of the right file. Locate your favorite editor with the online help and do the same. This also works for words in the Factor distribution, although it may be a bad idea to modify them.

This `\` word requires a little explanation. It works like a sort of escape, allowing us to put a reference to the next word on the stack, without executing it. This is exactly what we need, because `edit` is a word that takes words themselves as arguments. This mechanism is similar to quotations, but while a quotation creates a new anonymous function, here we are directly refering to the word `multiple?`.

Back to our task, you may notice that the words `[2,b]` and `multiple?` are just helper functions that you may not want to expose directly. To hide them from view, you can wrap them in a private block like this

    <PRIVATE

    : [2,b] ( n -- {2,...,n} ) 2 swap [a,b] ; inline

    : multiple? ( a b -- ? ) swap divisor? ; inline

    PRIVATE>

After making this change and refreshed the vocabulary, you will see that the listener is not able to refer to words like `[2,b]` anymore. The `<PRIVATE` word works by putting all definitions in the private block under a different vocabulary, in our case `github.tutorial.private`.

It is still possible to refer to words in private vocabularies, as you can confirm by searching for `[2,b]` in the online help, but of course this is discouraged, since people do not guarantee any API stability for private words. Words under `github.tutorial` can refer to words in `github.tutorial.private` directly, like `prime?` does.

Tests and documentation
-----------------------

This is a good time to start writing some unit tests. You can create a skeleton with

    "github.tutorial" scaffold-tests

You fill find a generated file under `work/github/tutorial/tutorial-tests.factor`. Notice the line

    USING: tools.test github.tutorial ;

that imports the unit testing module as well as your own. We will only test the public `prime?` function.

Tests are written using the `unit-test` word, which expects two quotations: the first one containing the expected outputs and the second one containing the words to run in order to get that output. Add these lines to `github.tutorial-tests`:

    [ t ] [ 2 prime? ] unit-test
    [ t ] [ 13 prime? ] unit-test
    [ t ] [ 29 prime? ] unit-test
    [ f ] [ 15 prime? ] unit-test
    [ f ] [ 377 prime? ] unit-test
    [ f ] [ 1 prime? ] unit-test
    [ t ] [ 20750750228539 prime? ] unit-test

You can now run the tests with `github.tutorial` test. You will see that we have actually made a mistake, and pressing `F3` will show more details. It seems that our assertions fails for `2`.

In fact, if you manually try to run our functions for `2`, you will see that our defition of `[2,b]` returns `{ 2 }` for `2 sqrt`, due to the fact that the square root of two is less than two, so we get a descending interval. Try making a fix so that the tests now pass.

There are a few more words to test errors and inference of stack effects. `unit-test` suffices for now, but later on you may want to check `must-fail` and `must-infer`.

We can also add some documentation to our vocabulary. Autogenerated documentation is always available for user-defined words (even in the listener), but we can write some useful comments manually, or even add custom articles that will appear in the online help. Predictably, we start with

    "github.tutorial" scaffold-docs

The generated file `work/github/tutorial-docs.factor` import `help.markup` and `help.syntax`. These two vocabularies define words to generate documentation. The actual help page is generated by the `HELP:` parsing word.

The arguments to `HELP:` are nested array of the form `{ $directive content... }`. In particular, you see here the directives `$values` and`$description`, but a few more exist, such as `$errors`, `$examples` and `$see-also`.

Notice that the type of the output `?` has been inferred to be boolean. Change the first lines to look like

    USING: help.markup help.syntax kernel math ;
    IN: github.tutorial

    HELP: prime?
    { $values
        { "n" fixnum }
        { "?" boolean }
    }
    { $description "Tests if n is prime. n is assumed to be a positive integer." } ;

and refresh the `github.tutorial` vocabulary. If you now look at the help for `prime?`, for instance with `\ prime? help`, you will see the updated documentation.

You can also render the directives in the listener for quicker feedback. For instance, try writing

    { $values
        { "n" integer }
        { "?" boolean }
    } print-content

The help markup contains a lot of possible directives, and you can use them to write stand-alone articles in the help system. Have a look at some more with `"element-types" help`.

The object system and protocols
-------------------------------

Although it is not apparent from what we have said so far, Factor has object-oriented features, and many core words are actually method invocations. To better understand how objects behave in Factor, a quote is in order:

> I invented the term Object-Oriented and I can tell you I did not have C++ in mind.

> Alan Kay

The term object-oriented has as many different meanings as people using it. One point of view - which was actually central to the work of Alan Kay - is that it is about late binding of function names. In Smalltalk, the language where this concept was born, people do not talk about calling a method, but rather sending a message to an object. It is up to the object to decide how to respond to this message, and the caller should not know about the implementation. For instance, one can send the message `map` both to an array and a linked list, but internally the iteration will be handled differently.

The binding of the message name to the method implementation is dynamic, and this is regarded as the core strenght of objects. As a result, fairly complex systems can evolve from the cooperation of independent objects who do not mess with each other internals.

To be fair, Factor is very different from Smalltalk, but still there is the concept of classes, and generic words can defined having different implementations on different classes.

Some classes are builtin in Factor, such as `string`, `boolean`, `fixnum` or `word`. Next, the most common way to define a class is as a tuple. Tuples are defined with the `TUPLE:` parsing word, followed by the tuple name and the fields of the class that we want to define, which are called 'slots' in Factor parlance.

Let us define a class for movies:

    TUPLE: movie title director actors ;

This also generates setters `>>title`, `>>director` and `>>actors` and getters `title>>`, `director>>` and `actors>>`. For instance, we can create a new movie with

    movie new "The prestige" >>title
      "Christopher Nolan" >>director
      { "Hugh Jackman" "Christian Bale" "Scarlett Johansson" } >>actors

We can also shorten this to

    "The prestige" "Christopher Nolan"
    { "Hugh Jackman" "Christian Bale" "Scarlett Johansson" }
    movie boa

The word `boa` stands for 'by-order-of-arguments' and is a constructor that fills the slots of the tuple with the items on the stack in order. `movie boa` is called a 'boa constructor', a word play on the Boa Constrictor. It is customary to define a most common constructor called `<movie>`, which in our case could be simply

    : <movie> ( title director actors -- movie ) movie boa ;

In other cases, you may want to use some defaults, or compute some fields.

The functional minded will be worried about the mutability of tuples. Actually, slots can be declared to be read-only with `{ slot-name read-only }`. In this case, the field setter will not be generated, and the value must be set a the beginning with a boa constructor. Other valid slot modifiers are `initial:` - to declare a default value - and a class word, such as `integer`, to restrict the values that can be inserted.

As an example, we define another tuple class for rock bands

    TUPLE: band { keyboards string read-only } { guitar string read-only }
      { bass string read-only } { drums string read-only } ;
    : <band> ( keyboards guitar bass drums -- band ) band boa ;

together with one instance

	"Richard Wright" "David Gilmour" "Roger Waters" "Nick Mason" <band>

Now, of course everyone knows that the star in a movie is the first actor, while in a rock band it is the bass player. To encode this, we first define a generic word

    GENERIC: star ( item -- star )

As you can see, it is declared with the parsing word `GENERIC:` and declares its stack effects but it has no implementation right now, hence no need for the closing `;`. Generic words are used to perform dynamic dispatch. We can define implementations for various classes using the word `M:`

    M: movie star actors>> first ;
    M: band star bass>> ;

If you write `star .` two times, you can see the different effect of calling a generic word on instances of different classes.

Builtin and tuple classes are not all that there is to the object system: more classes can be defined with set operations like `UNION:` and `INTERSECTION:`. Another way to define a class is as a mixin.

Mixins are defined with the `MIXIN:` word, and existing classes can be added to the mixin writing

    INSTANCE: class mixin

Methods defined on the mixin will then be available on all classes that belong to the mixin. If you are familiar with Haskell typeclasses, you will recognize a resemblance, although Haskell enforces at compile time that instance of typeclasses implent certain functions, while in Factor this is informally specified in documentation.

Two important examples of mixins are `sequence` and `assoc`. The former defines a protocol that is available to all concrete sequences, such as strings, linked lists or arrays, while the latter defines a protocol for associative arrays, such as hashtables or association lists.

This enables all sequences in Factor to be acted upon with a common set of words, while differing in implementation and minimizing code repetition (because only few primitives are needed, and other operations are defined for the `sequence` class). The most common operations you will use on sequences are `map`, `filter` and `reduce`, but there are many more - as you can see with `"sequences" help`.

Learning the tools
------------------

A big part of the productivity of Factor comes from the deep integration of the language and libraries with the tools around them, which are embodied in the listener. Many functions of the listener can be used programmatically, and viceversa. You have seen some examples of this:

* the help is navigable online, but you can also invoke it with `help` and print help items with `print-content`;
* the `F2` shortcut or the words `refresh` and `refresh-all` can be used to refresh vocabularies from disk while continuing working in the listener;
* the `edit` word gives you editor integration, but you can also click on file names in the help pages for vocabularies to open them.

The refresh is actually quite smart. Whenever a word is redefined, words that depend on it are recompiled against the new defition. You can check by yourself doing

    : inc ( x -- y ) 1 + ;
    : inc-print ( x -- ) inc . ;
    5 inc-print

and then

    : inc ( x -- y ) 2 + ;
    5 inc-print

This allows you to always keep a listener open, improving your definitions, periodically saving your definitions to file and refreshing, without ever having to reload Factor.

You can also save the whole state of Factor with the word `save-image` and later restore it by starting Factor with

    ./factor -i=path-to-image

In fact, Factor is image-based and only uses files when loading and refreshing vocabularies.

The power of the listener does not end here. Elements of the stack can be inspected by clicking on them, or by calling the word `inspector`. For instance try writing

    TUPLE: trilogy first second third ;
    : <trilogy> ( first second third -- trilogy ) trilogy boa ;
    "A new hope" "The Empire strikes back" "Return of the Jedi" <trilogy>
    "George Lucas" 2array

You will get an item that looks like

    { ~trilogy~ "George Lucas" }

on the stack. Try clicking on it: you will be able to see the slots of the array and focus on the trilogy or on the string by double-clicking on them. This is extremely useful for interactive prototyping. Special objects can customize the inspector by implementing the `content-gadget` method.

There is another inspector for errors. Whenever an error arises, it can be inspected with `F3`. This allows you to investigate exceptions, bad stack effects declarations and so on. The debugger allows you to step into code, both forwards and backwards, and you should take a moment to get some familiarity with it.

Another feature of the listener allows you to benchmark code. As an example, we write an intentionally inefficient Fibonacci:

    DEFER: fib-rec
    : fib ( n -- f(n) ) dup 2 < [ ] [ fib-rec ] if ;
    : fib-rec ( n -- f(n) ) [ 1 - fib ] [ 2 - fib ] bi + ;

(notice the use of `DEFER:` to define two mutually recursive words). You can benchmark the running time writing `40 fib` and then pressing Ctrl+T instead of Enter. You will get timing information, as well as other statistics. Programmatically, you can use the `time` word on a quotation to do the same.

You can also add watches on words, to print inputs and outputs on entry and exit. Try writing

    \ fib watch

and then run `10 fib` to see what happens. You can then remove the watch with `\ fib reset`.

Another very useful tool is the `lint` vocabulary. This scans word definitions to find duplicated code that can be factored out. As an example, let us define


images

Metaprogramming
---------------

macros, parsing words

When the stack is not enough
----------------------------

locals, fried quotations, global variables

Input/Output
------------

Deploying programs
------------------

factor scripts, deploy tool

Multithreading
--------------

Servers and Furnace
-------------------

Processes and channels
----------------------