# Semantics

As a **Haskell** *enthusiast*, I’m often asked by friends questions like *“how
do you do, in Haskell, to make things loop; to make things change”*. They ask
that because they’ve been told that *“in Haskell, everything is immutable”*.
From the point of view of someone who’s never tried a *Functional Programming
Language* (FPL), that can be, indeed, eh, disturbing.

If you think you fit in that category of people, keep reading. I’ll try to make
things a bit clearer.

## Purity and impurity

The first important thing to know about **Haskell** is the fact that, unlike
what you might have heard, **Haskell** is not about writing pure code – i.e.
with no *side-effects*, hence no pointers or whatever from imperative languages.
**Haskell** is more about balancing *side-effects* and *pure* code. That’s very
important. If something *requires* side-effect, why the hell would we go in
the opposite direction?

For instance, writing a line to screen requires a side-effect. We need to pass
a `String` to an opaque function that will mutate or act on
*something* – actually, on `stdout`, but it’ll require interaction with
something from the *outside*. Keep in mind that if we don’t have side-effects,
it’s like writing on a paper. We can prove anything we want, we can build hyper
elegant piece of software. But such software will be *useless*. Without
*side-effects*, no *interaction* can exist. We cannot provide inputs to our
programs. We cannot provide feedback to the user, through the screen or any
output device like a controlled robotic arm.

However, the good point about **Haskell**’s design is that it clearly put apart
*required mutation* and *unneeded mutation*. In order to get a line of input
via `stdin`, you need mutation. That kind of mutation is wrapped in an opaque
type in **Haskell**, called `IO`. `IO` is a polymorphic type. Its parameter
represent a value that *could* live somewhere else. For instance, consider the
following function:

```haskell
getLine :: IO String
```

That function tells us that it may require to perform some *side-effects* in
order to return a `String`. That `String` cannot be used directly, because we
need to *extract it* from `IO String`. The single way to do that is through
*monadic* code. I won’t explain what a *monad* is – there’re plenty of code
tutorials about that, check out the **Haskell** resource and keep away from
wikipedia; trust me, way too mathematic for you – but the core idea is pretty
simple. The single way to manipulate that `String` is to provide a function
that will take `String` and will output a new `IO a`. Here, `a` means
*anything*. In **Haskell**, that `a` value has no meaning, because it’s an
unconstrained *typevariable*. You can replace it with what you want. In our
case, we’ll use `putStrLn` as function. Consider its signature:

```haskell
putStrLn :: String -> IO ()
```

That’s a perfect match! It takes a `String`, and outputs `IO ()`. Here, `()` is
the **Haskell** type for the *unit* type. That is, a type with no information.
`putStrLn str` prints `str` on `stdout`, and returns *no information*.

Ok, now we can glue them together with the *bind* operator, `(>>=)`:

```haskell
getLine >>= putStrLn
getLine >>= \str -> putStrLn str
```

Those two lines are the same thing. The `String` from `getLine :: IO String`
gets extracted, then passed the next function, which is
`putStrLn :: String -> IO ()`. The resulting type of that expression is the
following ::

```haskell
getLine >>= putStrLn :: IO ()
```

You cannot get things out of `IO`. That means, you cannot call `IO` code
from pure code. That’s actually obvious, because the only thing we have to glue
`IO` functions is the bind `(>>=)` operator, that enforces the result to be in
`IO`.

That’s a very important thing to keep in mind. **Haskell** doesn’t forbid
*side-effects*. It only *isolates* them at the type level.

## Everything is not immutable

`IO` is a convenient type to represent side-effects. In `IO`, we can then
represent a set of *references*. They might be handy in certain cases, but in
general, we try to avoid them the most as possible. The most simple and most
known is `IORef`. There are a few functions about `IORef`:

```haskell
newIORef :: a -> IO (IORef a)
readIORef :: IORef a -> IO a
writeIORef :: IORef a -> a -> IO ()
modifyIORef :: IORef a -> (a -> a) -> IO ()
```

There are a few more functions, but we’ll stick to those for simplicity.
Basically, `newIORef x` creates a new `IORef a` and sets it to `x`, which must
have the type `a`. For instance:

```haskell
foo <- newIORef ""
```

`foo` is then an `IORef String`. Consider that code a bit like the following
**C++** snippet:

```
std::string *foo = new std::string;
```

