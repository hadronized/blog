# Foreword

Nowadays, there is a cool concept out there in the *Functional Programming*
world which is called *FRP*. It stands for *Functional Reactive Programming*
and is a pretty decent way to make *event-driven* programs.

The problem with FRP is that, beside [Wikipedia](http://en.wikipedia.org/wiki/Functional_reactive_programming),
[Haskell.org](https://wiki.haskell.org/Functional_Reactive_Programming) and a
few of other resources, like [Conal Elliott push-pull](http://conal.net/papers/push-pull-frp/)
papers, we lack learning materials. Getting into FRP is really not trivial
and because of the lack of help, you’ll need to be brave to get your feet
wet.

Because I suffered **a lot** learning it from scratch and because I think
it’s a good thing to pass knowledge by, I decided to write a few about it so
that people can learn via a simplest path.

I’ll be talking about [netwire](https://hackage.haskell.org/package/netwire),
which is not the *de-facto* library to use in Haskell, because eh… we don’t
have any yet. However, netwire exposes a lot of very interesting concepts and
helped me to understand more general abstractions. I hope it’ll help you as
well.

# The FRP Thing

## Event-driven programming context

In traditional event-driven programming codebase, you’d find constructions
such as events polling (when you explicitely retrieve last occurring events),
callbacks and *event handlers*. [GLFW](http://www.glfw.org/) is a very famous
example of callback uses for event-driven programming. Such functions as
[glfwSetWindowCloseCallback](http://www.glfw.org/docs/latest/group__window.html#gaade9264e79fae52bdb78e2df11ee8d6a)
require you to pass a callback that will be called when the event occurs. While
that way to go seems nice, it’s actually error-prone and ill-formed design:

  - you eventually end up with [spaghetti code](http://en.wikipedia.org/wiki/Spaghetti_code)
  - debugging callbacks is a true nightmare as the codebase grows
  - because of the first point, the code doesn’t compose – or in very minimal
    ways – and is barely impossible to test against
  - you introduce side-effects that might introduce nasty bugs difficult to
    figure out

![](http://phaazon.net/pub/spaghetti_code.jpg)

However, it’s not black or white. Callbacks are mandatory. They’re useful, and
we’ll see that later on.

## What FRP truely is?

FRP is a new way to look at event-driven programming. Instead of representing
reaction through callbacks, we consume events over time. Instead of building a
callback that will be passed as reaction to the `setPushButtonCallback`
function, we consume and transform events over time. The main idea of FRP could
be summed up with following concepts:

  - *behaviors*: a behavior is a value that reacts to time
  - *events*: events are just values that have occurrences in time
  - *switching*: the act of changing of behavior

### Behaviors

[According to Wikipedia](http://en.wikipedia.org/wiki/Behavior), *a behavior is the
range of actions and mannerisms made by individuals, organisms, systems, or artificial
entities in conjunction with themselves or their environment, which includes the other
systems or organisms around as well as the (inanimate) physical environment*. If we try
to apply that to a simple version of FRP that only takes into account the time as
external stimulus, it’s any kind of value that consumes time[^1]. What’s that? Well…

    newtype Behavior a = Behavior { stepBehavior :: Double -> a }

A behavior is a simple function from time (`Double`) to a given value. Let’s take an
example. Imagine you want to represent make a cube rotating around the *X axis*. You
can represent the actual rotation with a `Behavior Rotation`, because the angle of
rotation is linked to the time:

    rotationAngle = Behavior $ \t -> rotationFromAngle xAxis t

