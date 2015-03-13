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
well. :)

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

    rotationAngle :: Behavior Float Rotation
    rotationAngle = Behavior $ \t -> rotate xAxis t

Pretty simple, see?! However, it’d would more convenient if we could chose the type
of time. We don’t really know what the time will be in the final application. It
could be the current UTC time, it could be an integral time (think of a stepped
discrete simulation), it could be the monotonic time, the system time, something
that we don’t even know. So let’s make our `Behavior` type more robust:

    newtype Behavior t a = Behavior { stepBehavior :: t -> a }

Simple change, but nice improvement.

That is the typical way to picture a *behavior*. However, we’ll see later that
the implementation is way different that such a naive one. Keep on reading.

### Events

An event is *something happening at some time*. Applied to FRP, an event is a pair
of time – giving the time of occurrence – and a carried value:

    newtype Event t a = Event { getEvent :: (t,a) }

For instance, we could create an event that yields a rotation of 90° around X
at 35 seconds:

    rotate90XAt35s :: Event Float Rotation
    rotate90XAt35s = Event (35,rotate xAxis $ 90 * pi / 180)

Once again, that’s the naive way to look at events. Keep in mind that events have
time occurrences and carry values.

### Behavior switch

You switch your behavior every time. Currently, you’re reading this paper, but you
may go grab some food, go to bed, go to school or whatever you like afterwards.
You’re already used to behavior switching because that’s what we do every day in
a lifetime.

However, applying that to FRP is another thing. The idea is to express this:

> *“Given a first behavior, I’ll switch to another behavior when a given event
> occurs.”*

This is how we express that in FRP:

    switch :: Behavior t a -> Event t (Behavior t a) -> Behavior t a

Let me decrypt `switch` for you.

The first parameter, a `Behavior t a`, is the
initial behavior. For instance, currently, you’re reading. That could be the
first behavior you’d pass to `switch`.

The second parameter, an `Event t (Behavior t a)`, is an event that yields a
new `Behavior t a`. Do you start to get it? No? Well then:

    `switch reading finished`

`reading` is the initial behavior, and `finished` is an event that occurs when
you’re done reading. `switch reading finished` is then a behavior that equals
to `reading` until `finished` happens. When it does, `switch reading finished`
extracts the behavior from the event, and uses it instead.

I tend to think `switch` is a bad name, and I like naming it `until`:

    reading `until` finished

Nicer isn’t it?! :)

### Stepping

Stepping is the act of passing the input – i.e. time `t` in our case – down
to the `Behavior t a` and extract the resulting `a` value. Behaviors are
commonly connected to each other and form a *reactive network*.

That operation is also often refered to as *reactimation* in certain
implementations, but is more complex than just stepping the world. You don’t
have to focus on that yet, just keep in mind the `reactimate` function. You
might come across it at some time.

# Before going on…

Everything you read in that paper until now was just pop-science so that you
understand the main idea of what FRP is. The next part will cover a more
decent and real-world implementation and use of FRP, especially efficient
implementations.

# Let’s build a FRP library!

The first common error a lot of programmers make is trying to write
algorithms or libraries to solve a problem they don’t even know. Let’s then
start with an example so that we can figure out the problem.

## Initial program

Let’s say we want to be able to control a camera with a keyboard:

  - `W` would go forward
  - `S` would go backward
  - `A` would left strafe
  - `D` would right strafe
  - `R` would go up
  - `F` would go down

Let’s write the `Input` type:

```haskell
data Input
  = W
  | S
  | A
  | D
  | R
  | F
    deriving (Eq,Read,Show)
```

Straight-forward. We also have a function that polls events from `IO`:

```haskell
pollEvents :: IO [Input]
pollEvents = fmap treatLine getLine
  where
    treatLine = concatMap (map fst . reads) . words
```

We use `[Input]` because we could have several events at the same time
(imagine two pressed keys). The function is using dumb implementation
in order to abstract over event polling. In your program, you won’t use
`getLine` but a function from [SDL](https://hackage.haskell.org/package/sdl2)
or similar.
