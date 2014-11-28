# In the previous episode…

This blog entry directly follows [this one](http://phaazon.blogspot.co.uk/2014/11/abstracting-over-shader-environment.html). 
Feel free to read it before going on.

# Foreword

Before anything else, I’d like to point out something changed from the latest blog entry. The type called `Chain` is now called 
`Pipeline`. It’s more concise and simpler to understand that way.

# Varyings

## Stages

*Shaders* are commonly the name we give to *shader stages*. A full true *shader* is actually a chain of stages. There’re several
kind of shader stages:

  - vertex stage;
  - tessellation evaluation stage (optional);
  - tesselattion control stage (optional);
  - geometry stage (optional);
  - fragment stage.

Each stage has a frequency[^frequency]. That is really important, and **ash** defines respective frequencies as follows:

```haskell
data Vert deriving (Eq,Show)
data TessEval deriving (Eq,Show)
data TessCtrl deriving (Eq,Show)
data Geo deriving (Eq,Show)
data Frag deriving (Eq,Show)
```

## Between stages…

The interface between two stages should be strictly defined. In **GLSL**, that is done through *varying* variables.
Historically, **GLSL** has used several keywords as type qualifier:

  - `varying`, of course – now deprecated on most recent **GLSL** versions;
  - `in` for input varyings;
  - `out` for output varyings.

In **ash**, we’ll be using the type `Varying i o a` to refer to varyings. `Varying i o a` represents a free variable of type
`a` that will be referred to as inputs in *shaders* with frequency `i` and as outputs in *shaders* with frequency `o`. Let’s
see how we create such varyings and how we convert them to expressions we can use in our common expressions.

# Creating varyings

Creating varyings is pretty straight-forward. You have to do that in `Pipeline`.

> *“Wait, what?! Wasn’t supposed to be done in a shader stage?! That’s how I do that in GLSL!”*

Yeah but that way to do could be enhanced. Because you’ll be using the same expression in two different stages, you need both
the expression to be equally the same. If you’d create your varyings in a shader stage, you could end up with incompatibility
between the input and output versions of the varying. That’s why you create a varying in the pipeline. You define **how it’s
intended to be used** a better way then.

Then, here it is:

```haskell
varying :: (CPU a) => i -> o -> Pipeline (Varying i o a)
```

You have to pass two frequencies to `varying`. Then, `varying Frag Vert` creates a varying you can use as output in a vertex
shader and input in a fragment shader. Pretty amazing! If you try to use such varying in a geometry shader for instance,
you’ll get a compilation error, because the frequencies won’t match.

**Note: the `CPU a` typeclass constraint is there to only allow ash-supported types to be used as varyings.**

# Using varyings

Imagine you have a varying `vertPosVarying` you’ve gotten this way in your pipeline:

```haskell
vertPosVarying <- varying Frag Vert
```

Because I should have done a graphic designer ~carreer~ study, here is an awful diagram showing the concept:

![](http://phaazon.net/pub/ash%20%E2%80%93%20Varying.png)

Now, let’s use that in a vertex shader and a fragment shader.

## Turning varyings into expressions

There’s two functions to turn a `Varying i o a` into `Expr a`:

```haskell
varIn  :: Varying i o a -> Shader i (Expr a)
varOut :: Varying i o a -> Shader o (Expr a)
```

`varIn` can be used to turn a varying into an expression that represents the input part.



[^frequency]: The frequency of a shader stage gives how often it’ll be invoked (or for each object it will).
