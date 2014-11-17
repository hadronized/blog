# In the previous episode…

This blog entry directly follows [this one](http://phaazon.blogspot.co.uk/2014/11/abstracting-over-shader-environment.html). Feel free to read it before going on.

# Foreword

Before anything else, I’d like to point out something changed from the latest blog entry. The type called `Chain` is now called `Pipeline`. It’s more concise and simpler to understand that way.

# Shaders

As said earlier, shaders are functions that have access to an environment. In this post, we’ll see two new aspects about shaders:

  - they compose;
  - they have a frequency.

## Composition

Since a shader is *functionish*, we could expect it to be composable.

> *What do you mean, composable?*

If a shader `vs` outputs something of type `x`, and an other shader `fs` takes `x` an outputs `y`, we could write:

    p(x) = fs(vs(x)) <=> p = fs . vs

That is called composition. We do **love** composition in Haskell. And more generally, composability[^composability].

What do we mean here is that `fs . vs` is a shader composition, hence a new shader. Say that to a graphics developer, they’ll laugh their head off. Because gluing two shaders yielding a new shader doesn’t really make any sense for them.

### Theory, reality…

However, in a strictly theoric way, it does. People’s minds are shaped by current technologies such as *OpenGL* or *DirectX* in which shaders are hardly composable. In *OpenGL* for instance, shaders don’t compose. They’re put in a pipeline and “play their roles”. That’s pretty stupid, because we **know** it’s somewhat a composition though. A vertex shader outputs are directly passed to a fragment shader inputs, for instance. Or a vertex shader outputs to a geometry shader that outputs to a fragment shader:

    pipeline(vertex_components) = frag_shader(geo_shader(vert_shader(vertex_components)))
    pipeline = frag_shader . geo_shader . vert_shader

So why OpenGL doesn’t do that? Well, OpenGL separates the GLSL code source of each shader. Each shader has a `main` function, a uniform interface, a varying interface… You can’t directly use shaders, you have to put them altogether to yield a *shader program*. So, in OpenGL, if you “compose” two *shaders*, you end up with a *program*, and you can’t do much more with then, which is a pity.

### What’s about Ash?

In Ash, shaders do compose! You can write two shaders, and compose them the regular way compose functions or any composable objects. For instance:

```
vs = …
fs = …
fsvs = fs . vs
```

We’ll see that later on.

## Frequency


# Embedding shaders into pipelines

Shaders – or shader stages – have their own type in Ash: `Shader`.



[^composability]: See http://en.wikipedia.org/wiki/Composability.
