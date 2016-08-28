[luminance-0.7.0](https://crates.io/crates/luminance/0.7.0) was released a few days ago and I
decided it was time to explain exactly what luminance is and what were the design choices I made.
After a very interesting talk with [nical](https://github.com/nical) about other rust graphics
frameworks (e.g. [gfx](https://crates.io/crates/gfx), [glium](https://crates.io/crates/glium),
[vulkano](https://crates.io/crates/vulkano), etc.), I thought it was time to give people some more
information about luminance and how to compare it to other frameworks.

# Origin

luminance started as [a Haskell package](https://hackage.haskell.org/package/luminance), extracted
from a *“3D engine”* I had been working on for a while called
[quaazar](https://github.com/phaazon/quaazar). I came to the realization that I wasn’t using the
Haskell garbage collector at all and that I could benefit from using a language without GC. Rust
is a very famous language and well appreciated in the Haskell community, so I decided to jump in and
learn Rust. I migrated luminance in a month or two. The mapping is described in
[this blog entry](http://phaazon.blogspot.fr/2016/04/porting-haskell-graphics-framework-to.html).

# What is luminance for?

I’ve been writing 3D applications for a while and I always was frustrated by how OpenGL is badly
designed. Let’s sum up the lack of design of OpenGL:

- *weakly typed*: OpenGL has types, but… it actually does not. `GLint`, `GLuint` or `GLbitfield` are
  all defined as *aliases* to primary and integral types (i.e. something like
  `typedef float GLfloat`). Try it with `grep -Rn "typedef [a-zA-Z]* GLfloat" /usr/include/GL`. This
  leads to the fact that *framebuffers*, *textures*, *shader stages*, *shader program* or even
  *uniforms*, etc. have the same type (`GLuint`, i.e. `unsigned int`). Thus, a function like
  `glCompileShader` expects a `GLuint` as argument, though you can pass a framebuffer, because it’s
  also represented as a `GLuint` – very bad for us. It’s better to consider that those are just
  untyped – :( – handles.
- *runtime overhead*: Because of the point above, functions cannot assume you’re passing a value of
  a the expected type – e.g. the example just above with `glCompileShader` and a framebuffer. That
  means OpenGL implementations have to check against *all* the values you’re passing as arguments to
  be sure they match the type. That’s basically **several tests for each call** of an OpenGL
  function. If the type doesn’t match, you’re screwed and see the next point.
- *error handling*: This is catastrophic. Because of the runtime overhead, almost all functions
  might set the *error flag*. You have to check the error flag with the `glGetError` function,
  adding a side-effect, preventing parallelism, etc.
- *global state*: OpenGL works on the concept of global mutation. You have a state, wrapped in a
  *context*, and each time you want to do something with the GPU, you have to change something in
  the context. Such a context is important; however, some mutations shouldn’t be required. For
  instance, when you want to change the value of an object or use a texture, OpenGL requires you to
  *bind* the object. If you forget to *bind* for the next object, the mutation will occurs on the
  first object. Side effects, side effects…

  The goal of luminance is to fix most of those issues by providing a safe, stateless and elegant
  graphics framework. It should be as low-level as possible, but shouldn’t sacrifice
  runtime performances – CPU charge as well as memory bandwidth.
