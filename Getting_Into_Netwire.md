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

In traditional event-driven programming codebase, you’d find constructions
such as events polling (when you get events through a function like pollEvents),
