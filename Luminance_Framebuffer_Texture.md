I’m happily suprised **Haskell** so many **Haskell** people
follow [luminance](https://github.com/phaazon/luminance)! First thing first, let’s
tell you about how it grows.

Well, pretty quickly! There’s – yet – no method to make actual renders, because I’m
still thinking how to implement some stuff (I’ll detail that below), but it’s going
toward the right direction!

# Framebuffers

Something that is almost done is the [framebuffer](https://www.opengl.org/wiki/Framebuffer_Object)
part. The main idea of *framebuffers* – in **OpenGL** – is supporting *offscreen renders*, so that
we can render to several framebuffers and combine them to create effects. Framebuffers are often
bound *textures*, used to pass the rendered information around, especially to shaders, or to get
the pixels through texture CPU reads.

The thing is… **OpenGL**’s *framebuffers* are tedious. You can have incomplete framebuffer if you
don’t attach textures with the right format, or to the wrong attachment point. That’s why the
*framebuffer* layer of **luminance** is there to solve that.

In **luminance**, a `Framebuffer rw c d` is a framebuffer with two formats. A *color* format, `c`,
and a *depth* format, `d`. If `c = ()`, then no color will be recorded. If `d = ()`, then no depth
will be recorded. That enables the use of *color-only* or *depth-only* renders, which are often
optimized by GPU. It also includes a `rw` type variable, which has the same role as for `Buffer`.
That is, you can have *read-only*, *write-only* or *read-write* framebuffers.

## Textures

The format types are used to know which textures to create and how to attach them. You don’t have
to do that. The textures are hidden from the interface so that you can’t mess with them. I still
need to find a way to provide some kind of access to the information they hold, in order to use
them in shaders for instance. I’d love to provide some kind of *monoidal* properties between
framebuffers – to mimick [gloss](https://hackage.haskell.org/package/gloss) `Monoid` instance
for its [Picture](https://hackage.haskell.org/package/gloss-1.9.2.1/docs/Graphics-Gloss-Data-Picture.html#t:Picture)
type, basically.

## Pixel format

The cool thing is the fact I unified pixel formats. *Textures* and *framebuffers* share the same
pixel format type (`Format t c`). Currently, they’re all phantom types, but I might unify them and
use `DataKinds` to promote them to the type-level. A format has two type variables, `t` and `c`.

`t` is the underlying type. Currently, it can be either `Int32`, `Word32` or `Float`. I might add
support for `Double` as well later on.

`c` is the chanel type. They’re basically five channel types:

- `CR r`, a red channel ;
- `CRG r g`, red and green channels ;
- `CRGB r g b`, red, green and blue channels ;
- `CRGBA r g b a`, red, green, blue and alpha channels ;
- `CDepth d`, a depth channel (special case of `CR`, for depths only).

The type variables `r`, `g`, `b`, `a` and `d` represents *channel sizes*. They’re currently
three kind of *channel sizes*:

- `C8`, for 8-bit ;
- `C16`, for 16-bit ;
- `C32`, for 32-bit.

Then, `Format Float (CR C32)` is a red channel, 32-bit float – OpenGL equivalent is `R32F`.
`Format Word32 (CRGB C8 C8 C16)` is a RGB channel with RG two 8-bit unsigned integer channels
and the blue one is a 16-bit unsigned integer channel.

Of course, if a pixel format doesn’t exist on the **OpenGL** part, you won’t be able to use it.
Typeclasses are there to enforce the fact pixel format can be represented on the **OpenGL** side.

# Next work

Currently, I’m working hard on how to represent vertex formats. That’s not a trivial task, because
we can send vertices to OpenGL as interleaved – or not – arrays. I’m trying to design something
elegant and safe, and I’ll keep you informed when I finally get something. I’ll need to find
an interface for the actual render command, and I should be able to release something!

Keep the vibe, and have fun hacking around!
