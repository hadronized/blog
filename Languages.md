# About language design

As a software developper, I have to juggle the programming languages every day.
You might already know that, but there’re **a lot of languages** out there.
People tend to think that each language should be used for a very specific
purpose. Even though that was a true statement decades ago, is that still
true nowadays?

As you want to start writing a website, a webservice, a GUI application, an
embedded piece of critical software, or a real time appealing rendering
engine, people around will advise you to use a dedicated programming language.
For website programming? Go with PHP, Javascript, Ruby or even Python.
A webservice? Well, quite the same thing, but add Java EE or C# to the
equation along with serialisation and transport languages, like XML, JSON and
YAML. A GUI application? C# with winforms, C/C++ with Qt or Gtk+ or Java with
Swing. You need to write an embedded program? That would be difficult to use
anything else other than C and assembly. A real time rendering engine?
Definitely C++.

> **Disclaimer: I might have forgotten some languages. Don’t take it literally
though. Just keep reading.

However, it’s also possible to use Javascript for desktop application. Think of
[GNOME 3](https://www.gnome.org/gnome-3/). For a few years, they’ve decided to
write the desktop applications and the window manager in Javascript.

We can also use C to write webserver. I wouldn’t recommend that though. Since
node.js, we can use Javascript to do a lot of things. Python can be used to
write [Blender scripts](http://www.blender.org/api/blender_python_api_2_74_5/).
Although Perl was created to deal with files, like log files and configuration
files, it’s now possible to write websites with Perl with the
[catalyst web framework](http://www.catalystframework.org/).

It’s clear that things have changed. Programming languages’ designs have been
reviewed to cover more and more purposes. There’re still very specific
languages, like [Elm](http://elm-lang.org/), but in general, languages tend to
target the *multi-purposes paradigm*.

The point of this thread is to give my opinion on those languages and why we
should pick one language instead of another for a given context and problem.
Be warned though: it’s not a language advocacy. It’s only thoughts about
language design.

# What is a (programming) language good for?

A language is a way to speak with someone else. It’s a support. For instance,
Assembly is a very low level language designed to speak with a computer. It
maps more-or-less abstract concepts to concrete instructions a computer knows
how to read, and perfectly understands. The C language is a higher level
language that maps higher abstract concepts, like functions, structures,
pointers or variables, to concrete instructions a computer knows.

The common pattern is that a programming language is designed to be an
efficient way to transform a thought, an abstract concept, into a lower
concept. That transformation **always** responds to a context, a situation.
We don’t write programs for fun. We write them to solve problems. Then, a
programming language is there to help us think about the problem in higher
constructs. Assembly adds a very minimal set of features over the computer’s
native language, which is called
[machine code](http://en.wikipedia.org/wiki/Machine_code). The C# adds very
high abstractions, like
[delegates](http://en.wikipedia.org/wiki/Delegate_(CLI)) and automatically
handled memory. Functional languages adds even higher concepts, such as
[folds](http://en.wikipedia.org/wiki/Fold_%28higher-order_function%29),
[higher-order functions](http://en.wikipedia.org/wiki/Higher-order_function)
or [currying](http://en.wikipedia.org/wiki/Currying).

I think I’m not wrong if I state that in order to be *good*, a programming
language should help us solve a problem. If we start struggling with the
language on things that are not related to initial problem, the language
may not be suited for the problem. Or it sucks.

However, I truely think there is a common *need* for all languages. The need
to prevent the programmer from making errors. Languages that are too soft
about that tend to compile programs that have bugs, no matter whether the
programmer is good or not.

# Errors, bugs and prevention

## Dynamic typing vs. static typing

*Dynamic typing* is the property of a language that defer the type
resolution to runtime. What does that mean? Well, if you write this:

```
x = getLine();
print(x);
```

in a dynamically typed language, the type of `x` can’t be inferred at
compile time. It will be detected as a `string`, for instance, when you
run the program. That implies that type safety cannot be enforced at
compile time.

Something like the following will compile but *might break* when we run
the program:

```
x = getLine();
print(x + 3);
```

It might break because, depending on what the language allows about implicit
casts, it could still work fine if `x + 3` gets transformed into `x + "3"`.
Or that `x` gets parsed into a number, then `x + 3` can be evaluated, and
transformed into a `string` so that `print` can uses it.

Woah. Do you get it? Such a simple snippet, many possibilities about it.

Now, *static typing* is the property of a language in which we can infer
the type of an expression at compile time. If you write this:

```
x <- getLine();
putStrLn(x);
```

We know that the `getLine` function returns a `String`. It cannot be anything
else. We also know that `putStrLn` takes a `String` and writes it on stdout.
That snippet doesn’t allow for any *ambiguities*.

Let’s try something else.

```
n <- getLine();
putStrLn(n + 3);
```

That program cannot compile, because we try to add 3 to a `String`.

A lot of people think that such a behavior is boring. That we should be able
to implicitely cast `n` into an `Int`, or 3 into a `String. Wait a minute.

Let’s test both the scenario based on the fact `n = "10"` and that we can
concatenate two `String`s with `+`:

  - `n` is implicitely cast into an `Int`, so we get `10 + 3`, which is `13`;
  - `3` is implicitely cast into a `String`, so we get `"10" + "3"`, which is
    `"103"`.

Depending on how casts are performed, the yielded result is not the same.

Such a property of a language is called *weak typing*. A language is
*weakly typed* if it allows for implicit casts, or allow inconsistency between
types, like changing the type of an expression. Perl is famous about that.
In Perl, a function can return an integer value *and* a boolean value *and* a
hash *and* an array, etc. That makes that function weakly typed and will
confuse people every now and then.
