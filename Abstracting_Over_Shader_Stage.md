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

    Shader Vert a (V3 Float :. b)

Let’s analyze that signature. `Shader f a b` represents a shader stage that consumes values of type `a` at frequency `f` and outputs values of type `b`. So a *vertex shader* consumes `a` and outputs `V3 Float :. c`. The input must represent vertex components. The output is split in two parts `V3 Float` and `b`. `V3 Float` represents the computed position of the processed vertex. `b`

## Tessellation stages

The tessellation evaluation and tessellation control stages won’t be covered in this paper.

##

The geometry stage is run for each primitives (`Prim` frequency). It’s an optional stage.

The final stage is always a fragment stage. It’s run for all the covered pixels by primitives on screen.

# Embedding shaders into pipelines

Shaders – or shader stages – have their own type in Ash: `Shader`.



[^frequency]: The frequency of a shader stage gives how often it’ll be invoked (or for each object it will).
