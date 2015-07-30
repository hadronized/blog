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

The format types are used to know which textures to create and how to attach them. You don’t have
to do that. The textures are hidden from the interface so that you can’t mess with them. I still
need to find a way to provide some kind of access to the information they hold, in order to use
them in shaders for instance.
