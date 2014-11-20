# In the previous episode…

This blog entry directly follows [this one](http://phaazon.blogspot.co.uk/2014/11/abstracting-over-shader-environment.html). Feel free to read it before going on.

# Foreword

Before anything else, I’d like to point out something changed from the latest blog entry. The type called `Chain` is now called `Pipeline`. It’s more concise and simpler to understand that way.

# Shader stages

*Shaders* are commonly the name we give to *shader stages*. A full *shader* is actually a chain of stages. There’re several kind of shader stages:

  - vertex stage;
  - tessellation evaluation stage (optional);
  - tesselattion control stage (optional);
  - geometry stage (optional);
  - fragment stage.

## Vertex stage

The vertex stage is always the entry stage of a shader. It’s run for all vertices. We can say it has a `Vert` frequency[^frequency]. In Ash, a vertex shader must have the following type:

    Shader (Expr (a :. b -> V3 Float :. c))

Let’s analyze that signature.

`Shader a` is the type used to represent a stage. `(a :. b)` represents a tuple expression. Tuple expressions are used to glue types. We can see that a vertex shader has a type implying a function. It’s because it is, a function! It takes a tuple and returns a special tuple, `V3 Float :. c`.

## Tessellation stages

The tessellation evaluation and tessellation control stages won’t be covered in this paper.

##

The geometry stage is run for each primitives (`Prim` frequency). It’s an optional stage.

The final stage is always a fragment stage. It’s run for all the covered pixels by primitives on screen.

# Embedding shaders into pipelines

Shaders – or shader stages – have their own type in Ash: `Shader`.



[^frequency]: The frequency of a shader stage gives how often it’ll be invoked (or for each object it will).
