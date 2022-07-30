This blog article is about a crate I have been working on for a while now: [shades]. That library is a _shading
library_, that is, it provides you with types, traits and functions to create [shaders]. A shader is just a piece of
_code_ that typically runs on your GPU, and is written in a language such as [GLSL] — or anything that can target
[SPIR-V] these days.

# Context & problem

The way it typically works is by writing the GLSL code in a hard-coded string, or a file that is dynamically loaded at
runtime by your application (some people even pre-compile GLSL into a binary form, but it doesn’t change the fact that
the binary has to be brought to the GPU at runtime). The overall implication of all this is that the shader is
_represented_ via an opaque `String` (or binary like `Vec<u8>`) in your application.

A couple of examples of interfaces to handle shaders, using [wgpu] and a crate of mine, [luminance]:

- [`wgpu::ShaderSource`](https://docs.rs/wgpu/0.12.0/wgpu/enum.ShaderSource.html)
- [`luminance::shader::ProgramBuilder::from_strings`](https://docs.rs/luminance/0.47.0/luminance/shader/struct.ProgramBuilder.html#method.from_strings)

As you can see, dealing with shader sources implies a string interface, because that’s basically code that needs to be
compiled by your GPU driver (done at runtime). That has several issues:

- There is no way to _inspect_ the content of the shader to use its inputs, outputs, environment variables, etc. That
  leads to a lot of code duplication, which is just `String` duplication all over the place.
- Your application might compile and run, you are not sure that your shaders are correctly written, so you have to write
  a runtime checker for your shader, that needs to go through all the shaders in use in your application. Tedious, and
  error-prone.
- You have to learn a shading language. Most of the time, [GLSL] should be enough, but [GLSL] is old, and not very fun
  to work with.
- You will have to write the same code over and over and over. For instance, in [GLSL], there is no definition of π,
  so you will have to write its definition whenever you need it. That will quickly lead you to write some combinator
  code to inject sub strings containing such definitions and it will be a nightmare.
- It’s very weakly typed. A `String` doesn’t allow you to be sure that shader is compatible with the graphics pipeline.
  Being the author of [luminance], that is a massive flaw to me. As you might know, the typing in [luminance] is pretty
  advanced (phantom typing, type families, type states, etc.), so being stuck with `String` is not a good situation.

# Introducing `shades`

I introduced [shades] a couple of years ago without going public about it. The reason for that was that it was not
completely ready for people to use. Of course you could have given a try (and fundamentally, it hasn’t changed much),
but the syntax was going to drive people off right away. The idea of [shades] is to write shaders directly in Rust.
Instead of using `String` to represent a shader stage, you use `Stage<Inputs, Outputs, Environment>`. That gives you a
first visible advantage: typing. In [shades], you don’t have to declare your inputs, outputs nor environment. For
instance, compare the following GLSL code with its [shades] counterpart:

```glsl
uniform float time;
uniform vec2 resolution;
```

Whenever a shader stage has to use `time` or `resolution`, they will have to declare those two lines at the top of their
sources. With [shades]:

```rust
#[derive(Debug)]
struct Env {
  time: f32,
  resolution: V2<f32>,
}

#[derive(Debug)]
struct EnvExpr {
  time: Expr<f32>,
  resolution: Expr<V2<f32>>,
}

impl Environment for Env {
  type Env = EnvExpr;

  fn env() -> Self::Env {
    EnvExpr {
      time: Expr::new_env(0),
      resolution: Expr::new_env(1),
    }
  }

  fn env_set() -> Vec<(u16, Type)> {
    vec![
      (0, <f32 as ToType>::to_type()),
      (1, <V2<f32> as ToType>::to_type()),
    ]
  }
}
```

When using a `Stage<_, _, Env>`, `time` and `resolution` will automatically be available, whenever you use them or not.
This is even more interesting with procedural macro support in [luminance] for instance:

```rust
// the Environment and Semantics derive macro will generate all the above code for you, as well as the type family
#[derive(Debug, Environment, Semantics)]
struct Env {
  time: f32,
  resolution: V2<f32>,
}
```

So yes, [shades] is a bit more verbose at the Rust level, but I think it’s worth it because shaders are checked at
compile-time (via `rustc`) and not at runtime anymore. Typing also allows a better composition and reusability.

Just to give you an idea, this is an old and outdated example taken from the repository to show you what it used to be:

```rust
let vertex_shader = StageBuilder::new_vertex_shader(
  |mut shader: StageBuilder<MyVertex, (), ()>, input, output| {
    let increment = shader.fun(|_: &mut Scope<Expr<f32>>, a: Expr<f32>| a + lit!(1.));

    shader.fun(|_: &mut Scope<()>, _: Expr<[[V2<f32>; 2]; 15]>| ());

    shader.main_fun(|s: &mut Scope<()>| {
      let x = s.var(1.);
      let _ = s.var([1, 2]);
      s.set(output.clip_distance.at(0), increment(x.clone()));
      s.set(
        &output.position,
        vec4!(input.pos, 1.) * lit![0., 0.1, 1., -1.],
      );

      s.loop_while(true, |s| {
        s.when(x.clone().eq(1.), |s| {
          s.loop_break();
          s.abort();
        });
      });
    })
  },
);

let output = shades::writer::glsl::write_shader_to_str(&vertex_shader).unwrap();
```

That snippet doesn’t do anything really logical; it’s just there to show you how things look like.

I wrote a couple of things with [shades], and even though I like it, I have to admit that the Rust API is pretty hard to
read, use and understand. I needed something better.

# The new age of `shades`: `shades-edsl`

Then it struck me: why would _that_ API need to be the one people use? It can remain public, but maybe we can build
something _above_ it to make it much easier to use. The initial idea of that API was to be an [EDSL], but Rust has a lot
of limitations (for instance, `Eq` methods, `Ord` methods, etc. return `bool`, while I would need them to return
`Expr<bool>`). So… I came to the realization that the right way to do this would be to write a _real_ EDSL: [shades-edsl].

The idea of [shades-edsl] is super simple: remove everything that would make [shades] usable directly by a human, and
build a procedural macro that transforms regular Rust code into the API from [shades]. Basically, generate the [shades]
code inside the code of a procedural macro, whicth is what humans will use.

The way it works is pretty straight-forward to understand: you write your shading code in a `shades! { … }` block.
Everything in that block is reinterpreted and replaced by [shades] symbols: builders, method calls, etc. etc.

For instance, the following:

```rust
let stage = shades! { VS |_input, _output, _env| {
  fn add(a: i32, b: i32) -> i32 {
    a + b
  }

  fn main() {
    // do nothing
  }
}};
```

Will be reinterpreted as:

```rust
let stage = shades::stage::StageBuilder::<_, _, _>::new_vertex_shader(…);
```

Where `…` is generated code (a lambda here). A couple of more example:

```rust
  // in a shades! block
  if cond {
    body_if
  } else {
    body_else
  }
```

is transformed into:

```rust
  __scope.when(cond, |__scope| {
    #body_if // body_if is also transformed
  }).or_else(|__scope| {
    #body_else
  });
```

# Feature list

- Constant declarations: the regular Rust `const` declarations work and declare constant items in the shading code.
- Function declarations: two kind of declarations are supported:
  - Any kind of functions declared with `fn` will create function declarations and return a function handle (so that you
    can call the function).
  - A function called `main` will not return a function handle, but instead will inject the function declaration and
    will return the `Stage`. It is the only way to produce a `Stage` so you have the certainty that a `Stage` contains
    the `main` function.

# How it works

[shades-edsl] is implemented as a procedural macros, `shades!`, which defines a _shader stage_. It’s basically a
monadic computation (using `shades::stage::StageBuilder` behind the scene). Once you write your `fn main()` function,
the builder simply generates the `Stage` object for you (using `StageBuilder::main_fun`). In terms of architecture, it’s
a pretty simple one:

1. I use the [syn] crate to parse the content of the `shades!` invocation (`TokenStream`).
2. I then _mutate_ the generated AST to optimize and inject some logic in your code.
3. I marshal the AST back to Rust by generating the calls to the [shades] API, and write the Rust code back to the
   `TokenStream`

## Parsing the Rust code

Parsing the Rust code uses [syn], a famous parsing crate used a lot by procedural macros. It was made to parse Rust-like
code, but it has some interesting use cases. I’m used to writing procedural macros in [luminance-derive] for instance
(`#[derive()]` annotations).

The idea is to take a Rust item and represent it as a data type. The important thing to know with [syn] is that it
supports nice error message reporting, so you are advised to store non-meaningful information, like braces, parens,
tokens, etc.

For instance:

```rust
let x: Type = 3;
```

We can represent such an item (a `let` declaration) with the following `struct`:

```rust
#[derive(Debug)]
pub struct VarDeclItem {
  let_token: syn::Token![let],
  name: syn::Ident,
  ty: Option<(syn::Token![:], syn::Type)>,
  assign_token: syn::Token![=],
  expr: syn::Expr,
  semi_token: syn::Token![;],
}
```

> `syn::Token!` is a type macro that resolves to the right type name for the given symbol.

Parsing in [syn] uses the `Parse` trait, and is more intimidating than hard to use. This is the whole parser for
`VarDeclItem`:

```rust
impl Parse for VarDeclItem {
  fn parse(input: syn::parse::ParseStream) -> Result<Self, syn::Error> {
    let let_token = input.parse()?;
    let name = input.parse()?;

    let ty = if input.peek(Token![:]) {
      let colon_token = input.parse()?;
      let ty = input.parse()?;
      Some((colon_token, ty))
    } else {
      None
    };
    let assign_token = input.parse()?;
    let expr = input.parse()?;
    let semi_token = input.parse()?;

    Ok(Self {
      let_token,
      name,
      ty,
      assign_token,
      expr,
      semi_token,
    })
  }
}
```

`ParseStream` is the current state of the input stream. As you can see with the type of `parse()`, it will return the
`Self` type or an error, so we can call `input.parse()?` inside the implementation if type inference knows what the type
is, and that the type implements `Parse`. In our case, `syn::Token![let]` and `syn::Ident` do implement `Parse`, so we
can parse them in two simple lines:

```rust
    let let_token = input.parse()?;
    let name = input.parse()?;
```

Then, we need to look ahead to check whether there is a semicolon (`:`). If there is, then we can parse `syn::Token![:]`
and `syn::Type`, as both already implement `Parse`. Otherwise, we simply assume `None` as type ascription:

```rust
    let ty = if input.peek(Token![:]) {
      let colon_token = input.parse()?;
      let ty = input.parse()?;
      Some((colon_token, ty))
    } else {
      None
    };
```

The rest is as simple as everything implements `Parse` as well:

```rust
    let assign_token = input.parse()?;
    let expr = input.parse()?;
    let semi_token = input.parse()?;
```

We have everything we need to return the parsed declaration:

```rust
    Ok(Self {
      let_token,
      name,
      ty,
      assign_token,
      expr,
      semi_token,
    })
```

Every Rust items are parsed like that. There are a couple of harder things. For instance, input can be split by parsing
matched delimiters, resulting in two disjoint inputs, or some items will have to be delimited with punctuation (a
token).

An example of disjoint input parsing while parsing `if` statements:

```rust
#[derive(Debug)]
pub struct IfItem {
  if_token: Token![if],
  paren_token: Paren,
  cond_expr: Expr,
  brace_token: Brace,
  body: ScopeInstrItems,
  else_item: Option<ElseItem>,
}

// …

impl Parse for IfItem {
  fn parse(input: syn::parse::ParseStream) -> Result<Self, syn::Error> {
    let if_token = input.parse()?;

    let cond_input;
    let paren_token = parenthesized!(cond_input in input);
    let cond_expr = cond_input.parse()?;

    let body_input;
    let brace_token = braced!(body_input in input);
    let body = body_input.parse()?;

    let else_item = if input.peek(Token![else]) {
      Some(input.parse()?)
    } else {
      None
    };

    Ok(Self {
      if_token,
      paren_token,
      cond_expr,
      brace_token,
      body,
      else_item,
    })
  }
}
```

Here, we use the `parenthesized!` macro that parses a matching pair of `()`. Once it has finished, the `input` has
advanced after the closing delimiter, but the second input (`cond_input` in our case) is scoped right after the opening
delimiter and right before the closing one. That allows us to parse “inside” the parenthesis, which is a weird interface
at first, but once you get used to it, it’s actually pretty smart and pretty fast. Notice how we parse the conditional
expression using `cond_input` and not `input` here:

```rust
    let cond_input;
    let paren_token = parenthesized!(cond_input in input);
    let cond_expr = cond_input.parse()?;
```

Same thing with `{}` and the `braced!` macro:

```rust
    let body_input;
    let brace_token = braced!(body_input in input);
    let body = body_input.parse()?;
```

An even more interesting example: parsing function definitions:

```rust
#[derive(Debug)]
pub struct FunDefItem {
  fn_token: Token![fn],
  name: Ident,
  paren_token: Paren,
  args: Punctuated<FnArgItem, Token![,]>,
  ret_ty: Option<(Token![->], Type)>,
  brace_token: Brace,
  body: ScopeInstrItems,
}
```

Here, `args` is a punctuated sequence of `FnArgItem`, delimited with `,`. The parser is as follows:

```rust
impl Parse for FunDefItem {
  fn parse(input: syn::parse::ParseStream) -> Result<Self, syn::Error> {
    let fn_token = input.parse()?;
    let name = input.parse()?;

    let args_input;
    let paren_token = parenthesized!(args_input in input);
    let args = Punctuated::parse_terminated(&args_input)?;

    let ret_ty = if input.peek(Token![->]) {
      let arrow_token = input.parse()?;
      let ret_ty = input.parse()?;
      Some((arrow_token, ret_ty))
    } else {
      None
    };

    let body_input;
    let brace_token = braced!(body_input in input);
    let body = body_input.parse()?;

    Ok(Self {
      fn_token,
      name,
      paren_token,
      args,
      ret_ty,
      brace_token,
      body,
    })
  }
}
```

The argument part uses `parenthesized!` to parse the all arguments definitions, and then, with a single argument parser
(identifier plus type: `ident: Type`), we can simply ask to parse many by using comma punctuation. This is done with
`parse_terminated`:

```rust
    let args_input;
    let paren_token = parenthesized!(args_input in input);
    let args = Punctuated::parse_terminated(&args_input)?;
```

Parsing with [syn] is a bit weird at first (I’m used to [nom] because of [glsl]), but in the end, I think it’s a pretty
funny and interesting interface. There is no combinator such as alternative parser selection, but that’s probably for
the best, because the _peek_ / _look ahead_ interface forces you to write fast parsers. I really like it.

## Mutating the AST

Mutating the AST is a top-down operation that consists in transforming types and expressions. The goal is to _lift_
items up the shades EDSL. A function such as:

```rust
fn add(a: i32, b: i32) -> i32
```

Needs to be _representable_ in the `shades` AST. For this reason, `a`, `b` and the return type have to change types.
Changing the type implies going from `T` to `Expr<T>`. That operation is done in a `mutate(&mut self)` method on each
syntax items parsed from the previous stage, and each of `mutate` invocation might call `mutate` on items they contain
(such as punctuated items, collections of instructions / statements, etc.).

The only interesting part to discuss here is probably how we can lift values in complex expressions such as
`a + (b * c)`. This is done with the concept of mutable visitors, defined in [syn] with the `visit-mut` feature gate.
A mutable visitor is simply an object that implements a trait (`VisitMut`), which defines one method for each type of
items in the AST that might exist. For instance, there is `visit_expr_mut(&mut self, e: &mut syn::Expr)` and a
`visit_type_mut(&mut self, e: &mut syn::Type)`. All `visit_*` methods have a default implementation doing nothing, so
that you are left with implementing only the ones that matter for you.

An example of what I do with types:

```rust
struct ExprVisitor;

impl VisitMut for ExprVisitor {
  fn visit_type_mut(&mut self, ty: &mut Type) {
    *ty = parse_quote! { shades::expr::Expr<#ty> };
  }
}
```

> `parse_quote!` is a [syn] macro that is akin to `quote!` from [quote], but can infer the return type to parse to the
> right type. Here, the returned type is inferred to be `syn::Type`.

An interesting thing with [shades-edsl]: because users will provide their inputs, outputs and environment types in the
types of `Stage`, we do not want to wrap those in `Expr`, because they already contain `Expr` values, as seen earlier.
For this reason, the syntax of types of arguments was augmented with the possibility to add a dash (`#`) in behind the
types. If such a symbol is found, the type is assumed _unlifted_ and shouldn’t be lift. For instance:

```rust
let a: i32 = …; // will become Expr<i32>
let b: #Foo = …; // will remain Foo
```

## Generating the `shades` code

Generating the [shades] AST is just a matter of traversing the parsed and mutated AST and using the [quote] crate and
its `quote!` macro to write the Rust code we want. The only missing part is how to bootstrap a `StageBuilder`, which is
done with some more parsing. Invocation looks like this:

```rust
  let stage = shades! { vertex |input: TestInput, output: TestOutput, env: TestEnv| {
```

The first keyword is `vertex` here, but it could be `fragment`, `geometry` or any kind of shader stage (that needs to be
documented). Then, you have a lambda. I chose that syntax because a shader stage can be imagined as a function of its
input, output and environment. I hesitated with returning the output, but it would make things harder for some stages
(i.e. geometry shaders), where they can write to the same output several times. In that lambda, every type is
automatically marked as unlifted (so `input: TestInput` and `input: #TestInput` are the same).

Once transformed, this single line is akin to something like this:

```rust
shades::stage::StageBuilder::new_vertex_shader(|mut __builder: shades::stage::StageBuilder::<#user_in_ty, #user_out_ty, #user_env_ty>, #input, #output| {
```

Everything else works the same way. Some fun example:

The generated code when the `main` function is found:

```rust
    if name.to_string() == "main" {
      let q = quote! {
        // scope everything to prevent leaking them out
        {
          // function scope
          let mut __scope: shades::scope::Scope<()> = shades::scope::Scope::new(0);

          // roll down the statements
          #body

          // create the function definition
          use shades::erased::Erased as _;
          let erased = shades::fun::ErasedFun::new(Vec::new(), None, __scope.to_erased());
          let fundef = shades::fun::FunDef::<(), ()>::new(erased);
          __builder.main_fun(fundef)
        }
      };

      q.to_tokens(tokens);
      return;
    }
```

If items:

```rust
impl ToTokens for IfItem {
  fn to_tokens(&self, tokens: &mut proc_macro2::TokenStream) {
    let cond = &self.cond_expr;
    let body = &self.body;

    let q = quote! {
    __scope.when(#cond, |__scope| {
      #body
    }) };

    q.to_tokens(tokens);

    match self.else_item {
      Some(ref else_item) => {
        let else_tail = &else_item.else_tail;
        quote! { #else_tail }.to_tokens(tokens);
      }

      None => {
        quote! { ; }.to_tokens(tokens);
      }
    }
  }
```

Else tail items, which allows to add an `if` again (`else if`) or stop to `else` only. You will notice in `ToTokens`
implementation of `IfItem` that there is no `;` added at the end of the `.when()` call unless there is no `else`
following, and that rule is kept enforced in `ToTokens for ElseTailItem` as well. See, static analysis at compile-time!:

```rust
impl ToTokens for ElseTailItem {
  fn to_tokens(&self, tokens: &mut proc_macro2::TokenStream) {
    match self {
      ElseTailItem::If(if_item) => {
        if_item.to_tokens_as_part_of_else(tokens);
      }

      ElseTailItem::Else { body, .. } => {
        let q = quote! {
          .or(|__scope| { #body });
        };

        q.to_tokens(tokens);
      }
    }
  }
}
```

# Writers

Once you get a `Stage`, you can use a _writer_ to output a `String`, so that [shades] is compatible with any kind of
library / framework / engine that expects a string-based shader source.

For example, for GLSL:

```rust
  let output = shades::writer::glsl::write_shader_to_str(&vertex_shader).unwrap();
```

# Interaction with `luminance`

Eh, I wrote both crates, so obviously [luminance] has / will have a special support of [shades]. Remember the definition
of `Environment` above? It’s similar to `Inputs` and `Outputs`, too. [luminance-derive] already provides some derive
macros to automatically implement some traits, and it will provide a support to implement `Inputs`, `Outputs` and
`Environment` as well, without having to think about the indexes.

# Caveats and alternative work

Of course, if something is not implemented correctly (a bug in [shades-edsl], a bug in [shades]), debugging the output
shaders will be harder, because everything is generated (names included) on the target side. It’s something to keep in
mind.

You should also have a look at some similar yet different approaches:

- [rust-gpu](https://github.com/EmbarkStudios/rust-gpu) from EmbarkStudios. It’s a similar project but instead of
  parsing Rust code, they wrote a compile chain (that you need to install via `rustup`) and compile your code with. It’s
  an interesting point but I wanted something more typed with a more seamless integration to the rest of my ecosystem
  (i.e. [luminance]).
- [wgsl](https://www.w3.org/TR/WGSL), a new shading language for the WebGPU.
- [glsl-quasiquote]: you can write GLSL code in it and get it parsed at compile-time. However, it will use the [glsl]
  crate, which is currently a parser only, and it will not do any kind of semantic analysis. Another crate has to be
  written for that, and it doesn’t exist to my knowledge (and I won’t write it because it’s a lot of work and I don’t
  think it’s the right approach, unless you want to write a GLSL compiler, which I don’t).

Keep the vibes!

[shades]: https://crates.io/crates/shades
[shades-edsl]: https://crates.io/crates/shades-edsl
[shaders]: https://en.wikipedia.org/wiki/Shader
[GLSL]: https://www.khronos.org/opengl/wiki/OpenGL_Shading_Language
[SPIR-V]: https://www.khronos.org/spir
[wgpu]: https://crates.io/crates/wgpu
[luminance]: https://crates.io/crates/luminance
[EDSL]: https://en.wikipedia.org/wiki/Domain-specific_language
[syn]: https://crates.io/crates/syn
[luminance-derive]: https://crates.io/crates/luminance-derive
[glsl-quasiquote]: https://crates.io/crates/glsl-quasiquote
[glsl]: https://crates.io/crates/glsl
[nom]: https://crates.io/crates/nom
[quote]: https://crates.io/crates/quote
