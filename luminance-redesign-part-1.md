Lately, I have decided to take a deep look at [luminance]. Issues, current code, frustration I had using it, feedback
from people using it, etc. I realized two things:

1. The first commit of [luminance] was made in 2016, as I migrated it away from its
  [Haskell version](https://hackage.haskell.org/package/luminance), which first commit was made in 2015. Since then, my
  knowledge of programming, both Haskell and Rust, but also in general has enhanced quite a lot.
2. The current design has some flaws, both in terms of usability and maintainability.

About the first point, I had a look at code that barely changed since I wrote it in 2016, and I’m not completely
satisfied with it anymore. Lots of things happened ever since: I have a sharper vision of how things should be done,
Rust itself has changed quite a lot (back in 2016, Rust 1.0 was just released!), we have more type system features that,
back then, I was lacking, etc.

About the second point, some issues in the current version of [luminance] (`v0.47` as writing this blog article) are
complex to solve, such as the [Drop problem](https://github.com/phaazon/luminance-rs/issues/304). Other parts of the API
are insanely complex, like the
[`TessBuilder`](https://docs.rs/luminance/0.47.0/luminance/tess/struct.TessBuilder.html#impl-5) related code.

I want all that to change, so I started working on a completely new graphics crate, just to see what kind of interface I
could come up with. And I came up with something much, much easier. So I faced a dilemma: should I release that other
crate and give up on [luminance]… or should I just port all the new knowledge and ideas from that crate into
[luminance]?

Well, I have decided to keep [luminance] around and update it. It is going to take time, because [luminance] is an
ecosystem of many crates, it has lots of examples (which is great to ensure that its features are still working
correctly once I start redesign it) and because some features are obviously entangled. However, I think it’s worth it,
because even though [luminance] is not as famous or widely used as [wgpu], it has a special place in the Rust Graphics
community.

So I’m going to blog about the redesign of [luminance] and it’s going to be a blog article series. I wanted to start
with a very new exciting feature that I have been wanting for a very long time: compatible vertex types.

# The vertex compatibility problem

When you’re like me and are obsessed with tracking as much information as possible at compile-time, you start imagining
lots of features and ways to _prevent_ people from doing things. Preventing and forbidding is actually much more complex
than allowing something. Think about it like this: assembly allows for everything and is not a complex language. It has
zero constructs, besides opcodes. However, as you add more constructs and abstractions, those abstractions _restricts_
the application domain; they forbid usage, in a constrained way. There’s a reason why, in Haskell, we use the term
`Contraint` — which is called, basically, _trait bound_ in Rust.

The way [luminance] allows a programmer to render objects, which must have a vertex type `V` that must implement
`Vertex`. [luminance-derive] eases the process of implementing lots of traits from [luminance]. For instance:

```rust
#[repr(C)]
#[derive(Clone, Debug, Vertex)] // < notice the Vertex derive here
pub struct MyVertex {
  position: Vector3<f32>,
  normal: Vector3<f32>,
  color: Vector3<u8>,
  weight: Vector3<f32>,
}
```

This is great, because the user just has to focus on writing types and just using them. For vertex objects, which can be
imagined as big arrays / buffers of `MyVertex` here, there are types and functions allowing to basically render such
object, and those render functions expects `MyVertex`. However, rendering is a multi-stage process. As you may know, the
graphics pipeline contains different steps, and among the first ones, you have a vertex shader, which is responsible in
mapping vertices (to transform them, for instance). In [luminance], vertex shaders are strongly-typed with the kind of
vertices they expect. That is to prevent a user from using a vertex shader that expects some vertex attributes that a
vertex object doesn’t have. Imagine that we write a vertex shader that expects a `position` and a `normal` but our
vertex object, which uses `OtherVertex` vertices, only has `position`. That would result in probably some visual
glitches or just a black screen.

So, we want to have something like this pseudocode:

```rust
type VertexShader<V>; // accept only vertex type V
type RenderVertexObjects<W>; // render only vertex type W
```

If we want to have a fully typed graphics pipeline, we must have `V = W`. However, that is going to be very annoying.
Imagine a vertex shader that only needs `position`, `normal` and `color`. Something like:

```rust
#[repr(C)]
#[derive(Clone, Debug, Vertex)]
pub struct ShaderVertexInput {
  position: Vector3<f32>,
  normal: Vector3<f32>,
  color: Vector3<u8>,
}
```

This is a different type than `MyVertex`, so if we try to render our `MyVertex` vertex objects with a vertex shader that
works on `ShaderVertexInput`, it will get a type mismatch at compile-time. The only way to fix that is to create a copy
of the shader program that works with `MyVertex` and ignore the `weight` field. Meh. Waste of resources and duplication.
Everything we don’t want.

When you look at it, the vertex shader is going to use `position`, `normal` and `color`, but it doesn’t have to know
there are other vertex attributes. So it should be able, in theory, to work with `MyVertex`, since `MyVertex` has
`position`, `normal` and `color`…

# The solution

One solution to fix the problem is to introduce a trait, `CompatibleVertex`, that we will use on the vertex shader code.
The vertex shader still has its `ShaderVertexInput` type, but it will not require vertex objects to use that type.
Instead, it will require vertex objects to use `V` and will require `ShaderVertexInput: CompatibleVertex<V>`. What it
means is that `ShaderVertexInput` must be _included_ in `V`. Said otherwise, `V` must have all of `ShaderVertexInput`’s
fields (but it can have more).

```rust
pub trait CompatibleVertex<V> {}
```

Now the big question: how do we implement `CompatibleVertex` for a given concrete vertex type? Well, in the previous
version of [luminance], that used to be implemented manually by users of [luminance]. It was both unsafe and not really
practical, so I just removed the concept (and [luminance] in version `v0.47` and less is subject to the problem I
described above, where you can render a vertex object which vertex type is not compatible with what the shader expects).

## Const generics to the rescue

Since `rustc-1.51`, const generics can be used at compile-time. The feature allows to manipulate integers at
compile-time. For instance:

```rust
pub struct Array<const N: usize> {
  // …
}
```

Here, `N` is a const generics, and you use it with types like `Array<3>` or `Array<14>`. However, that’s currently
(`rust-1.63`) all you can do with a stable compiler… but we can do much more with a nightly compiler.

There is a feature called `adt_const_params` that allows using more types, and one type that is very important is
`&'static str`. Yes, you heard right. Constant strings. We can write something like this:

```rust
#![allow(incomplete_features)] // as of rustc-1.65 nightly, this is still required
#![feature(adt_const_params)]

fn hello<const N: &'static str>() {
  println!("Hello {}!", N);
}

fn main() {
  hello::<"world">();
}
```

That doesn’t seem like much, but this small change is going to save so many frustration in [luminance], and fix our
compatible vertex problems.

## Applying adt_const_params to CompatibleVertex

What we basically want for one vertex type to compatible with another (i.e. one is _included_ into the other) is to
ensure that the all the fields of one are present in the other. Let’s recall our vertex type definitions and trait:

```rust
#[repr(C)]
#[derive(Clone, Debug, Vertex)]
pub struct MyVertex {
  position: Vector3<f32>,
  normal: Vector3<f32>,
  color: Vector3<u8>,
  weight: Vector3<f32>,
}

#[repr(C)]
#[derive(Clone, Debug, Vertex)]
pub struct ShaderVertexInput {
  position: Vector3<f32>,
  normal: Vector3<f32>,
  color: Vector3<u8>,
}

pub trait CompatibleVertex<V> {}
```

We can write a `HasField` trait like this:

```rust
pub trait HasField<const NAME: &'static str> {
  type FieldType;
}
```

And now, we can implement that trait for all the fields of our vertex types:

```rust
// for MyVertex
impl HasField<"position"> for MyVertex {
  type FieldType = Vector3<f32>;
}

impl HasField<"normal"> for MyVertex {
  type FieldType = Vector3<f32>;
}

impl HasField<"color"> for MyVertex {
  type FieldType = Vector3<u8>;
}

impl HasField<"weight"> for MyVertex {
  type FieldType = f32;
}

// for ShaderInputVertex
impl HasField<"position"> for ShaderInputVertex {
  type FieldType = Vector3<f32>;
}

impl HasField<"normal"> for ShaderInputVertex {
  type FieldType = Vector3<f32>;
}

impl HasField<"color"> for ShaderInputVertex {
  type FieldType = Vector3<u8>;
}
```

Now, we can write a more meaningful implementor for `CompatibleVertex` and `ShaderInputVertex`:

```rust
impl<V> CompatibleVertex<V> for ShaderInputVertex where
  V: HasField<"position", FieldType = Vector3<f32>>
    + HasField<"normal", FieldType = Vector3<f32>>
    + HasField<"color", FieldType = Vector3<u8>>
{
}
```

Because `MyVertex` implements the three `HasField` trait bounds required, it can be used, as well as
`ShaderInputVertex`.

Obviously, because [luminance-derive] is a thing, people will never have to write any of those implementors. Actually,
deriving `Vertex` will do two things (besides what it already does) for a given vertex type:

- Provide all the `HasField` implementors for all of its field.
- Provide the universal `CompatibleVertex` implementor.

What it means is that, as soon as you start using `#[derive(Vertex)]`, you will automatically have compatible vertex
types behind the scenes working for you.

# Conclusion

That feature is already, by itself, worth requiring a nightly compiler, as it brings the exact kind of type safety I
want for my code. And since I started with const generics strings, the next blog article will be about the redesign of
vertex objects (called `Tess` in the current version of [luminance]), since, you might have guessed, it will use const
generics strings to upload vertex data per-attribute, and map / update them.

Thanks for having read, and keep the vibe!

[luminance]: https://crates.io/crates/luminance
[luminance-derive]: https://crates.io/crates/luminance-derive
[wgpu]: https://crates.io/crates/wgpu
