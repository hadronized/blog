# Foreword

This paper is a pause from my spree about the **ash** shading language library I’m working on. I’m also working on a
3D engine, [photon](https://github.com/phaazon/photon), in which I plan to use **ash**. The projects are then closely
related ;).

The purpose of this paper is to discover a nice way of dealing with the *reaction* problem. There’re common and
elegant solutions that address the problem, like FRP[^FRP], which is pretty elegant, but also very experimental. I
wanted to explore on my own. This is what I come up with.

## Side effect

As a programmer, you might already have come across **side effects**. You haven’t? Let’s have a look at these nasty
things.

A **side effect** is a big word to describe an *effect* a function has that mutates something out of scope. For
instance, you could picture a function modifying a global state, environment or any kind of “*remote*” value.

Another way to understand what a side effect is is to look at
[referential transparency](http://en.wikipedia.org/wiki/Referential_transparency_%28computer_science%29). If you
need *assumptions* to be able to say what a piece of code does, you might be in the presence of a side effect.
For instance, look at this **C++** snippet:

```
void update(int rx, int ry) {
  _x += rx;
  _y += ry;
}
```

Do you know what those lines do?

> *Yeah, sure! It’s a method in a class that updates `_x` and `_y` fields by applying them offsets `rx` and `ry`!*

That could be. Let me add some more code to complete the example:

```
int _x = 0;
int _y = 0;

void update(int rx, int ry) {
  _x += rx;
  _y += ry;
}

int main() {
  update(2,1);
  return 0;
}
```

A *class*, you said? As you can see, we can’t say what that code does, because we have to look at **all possible**
code lines that code might touch. It could be a class method, it could be a simple function, `_x` and `_y` could be `int`
as well as they could be `Foo` objects with the `operator+=` overloaded. The list is long.

In that snippet, `_x += rx` and `_y += ry` are side effects, because they alter “something” *elsewhere*. That could
be fine, but it’s not. The more you have side effects, the harder it is to debug, maintain, compose, understand and
make your code base evolve. As a good programmer, you should care about side effect avoidance.

## Purity

In pure functional languages like **Haskell**, we can’t do that kind of assignment, at least not directly. We would write this function:

```
update :: (Int,Int) -> (Int,Int) -> (Int,Int)
update (x,y) (rx,ry) = (x + rx,y + ry)
```

Because everything is immutable in **Haskell**, we are sure that `(x,y)` and `(rx,ry)` are constant. They **can’t**
change. So we can say what that function does:

> *It takes a pair of int and apply them an offset!*

Yeah, exactly. And it can’t be **anything else**. We don’t need assumptions, because such a function is
*transparent*. That’s a great property you can rely on – do it, your compiler does ;)

## However, we still need side effects

If **Haskell** rejected all side effects, we couldn’t have any programs. We would have theorems, properties and
purity, but nothing on screen, no inputs, nothing actually computing. That would be a pity. But why? Well, we
still need side effects. When you print something on screen, you actually want a side effect. There’s nowadays
no other way to go. You pass a `String` to `putStrLn` for instance, and the function generates a side effect to
put the `String` on screen. That’s a side effect because it’s out of scope of your program. You touch something
elsewhere.

That’s the same thing for inputs, reading files, creating files and so on and so forth. **Haskell** uses a type
to be able to deal with those cases: `IO a`. It’s a polymorphic type that can look at the **real world**. You
can’t but `IO a` can.

We have `IO a` to explicitely deal with side effects, that’s cool. However, `IO` represents **any** kind of side
effects. We’d like to be able to explicitely say *“Hey, I can have that effect”*.

# Reaction: Part 1 – Explicit effects in imperative languages

Reacting to something requires activation. Do you know the
[observer design pattern](http://en.wikipedia.org/wiki/Observer_pattern)? That is an interesting design pattern
everyone should know. It’s very powerful since it enables you to run actions when an observed event is generated.
You have to explicitely describe what actions and/or objects you want to observe. That is commonly done via
two important things:

  - implementing an *interface* that exports the *event handlers* interface;
  - explicitely interleaving your observed code with calls to the abstract *event handlers*.

Let’s take an example, still in **C++**:

```
// this is the interface used to react to events
class FooObserver {
  FooObserver(void) {}
  virtual ~FooObserver() = 0;
  
  virtual void on_fire(Direction const &dir) = 0;
  virtual void on_set_a(int oldValue, int newValue) = 0;
  virtual void on_set_b(std::string const &oldValue, std::string const &newValue) = 0;
};

class Foo {
private:
  int _a;
  std::string _b;
  std::list<FooObserver&> _observers; // all observers that will react to events
  
public:
  Foo(void) : _a(0), _b("") {}
  ~Foo(void) {}
  
  void addObserver(FooObserver &observer) {
    _observers.push_back(observer);
  }
  
  void fire(Direction const &dir) {
    // use dir
    for (auto observer : _observers)
      observer.on_fire(dir); // notify all observers something has happenned
  }
  
  void setA(int a) {
    _a = a;
    for (auto observer : _observers)
      observer.on_set_a(_a, a); // notify all observers something has happenned
  }
  
  void setB(std::string const &b) {
    _b =  b;
    for (auto observer : _observers)
      observer_on_set_b(_b, b); // notify all observers something has happenned
  }
};
```

If we want to react to events emmitted by an object of type `Foo`, we just have to create a new type
that inherits from `FooObserver`, implement its abstract methods and register an object of our type
so that the value can call it when it has to emmit events. That’s pretty great, but it has a lot of
side effects, and we’re gonna try to abstract that away.

# Reaction: Part 2 – Explicit effects in **Haskell**

I’ve been wondering around for a while. There’re folks that advise to use 
[FRP](https://www.haskell.org/haskellwiki/Functional_Reactive_Programming). It addresses the issue
another way though – I won’t talk about FRP in this post, maybe later since it’s a very interesting
concept. For my part, I wanted something like [pipes](http://hackage.haskell.org/package/pipes).
Being able to compose my functions along with having effects.

In [photon](https://github.com/phaazon/photon), my 3D engine, I use explicit effects to implement
reactions. That is done via a typeclass called `Effect`:

```haskell
class (Monad m) => Effect e m where
  react :: e -> m ()
```

Pretty straight-forward eh? We call `react e` to react to an event of type `e`. Let’s have a look
at a few examples.

## `react`, examples – Part 1

Let’s start with a simple example:

```haskell
-- This type represents all effects we want to observe.
data IntChanged
  = IntSucc
  | IntPred
  | IntConst Int
    deriving (Eq,Show)

-- This instance enables us to react in a State Int.
instance  Effect IntChanged (State Int) where
  react e = case e of
    IntSucc -> modify succ
    IntPred -> modify pred
    IntConst x -> put x

foo :: (Effect IntChanged m) => m String
foo = do
  react (IntConst 314)
  return "foo"

bar :: (Effect IntChanged m) => Float -> m Float
bar a = do
  when (sqrt a < 10) . replicateM_ 3 $ react IntSucc
  return (a + pi)
```

Let’s use that. I use explicit types because I’m in *ghci*:

> `flip runState 0 (foo :: State Int String)`

> ("foo",314)

> `flip runState 0 (bar 0 :: State Int Float)`

> (0.0,3)

> `flip runState 0 (bar 10 :: State Int Float)`

> (3.1622777,3)

> `flip runState 0 (bar 99 :: State Int Float)`

> (9.949874,3)

> `flip runState 0 (bar 100 :: State Int Float)`

> (10.0,0)

> `flip runState 0 (bar 314 :: State Int Float)`

> (17.720045,0)

As you can see, we can have effects without `IO`. In that case, it was pretty simple. But since
it’s abstract to any `Monad`, we could implement effects in `IO`, specific ones.

## `react`, examples – Part 2

Let’s see an example in `IO`.

> `foo`

> set to 314

> "foo"

> `bar 0`

> succ!

> succ!

> succ!

> 0.0

> `bar 10`

> succ!

> succ!

> succ!

> 3.1622777

> `bar 99`

> succ!

> succ!

> succ!

> 9.949874

> `bar 100`

> 10.0

> `bar 314`

> 17.720045

Because our `foo` and `bar` functions are polymorphic, we can use them with any types
implementing the wished effects! That’s pretty great because it enables us to write our code in an
abstract way, and interpret it with backends.

# Extra – handles

Because all of this was firstly designed for my **photon** engine, I had to deal with an important question.
Having *effects* is great, but how could we make an effect like:

> *“Draw the mesh with ID=486.”*

## Handles

I use *handles* to deal with that. I use a type to represent handles (`H`). Each object that can be *managed*
(i.e. that can have a handle) can be wrapped up in `Managed a`. Basically:

```haskell
type H = Int

data Managed a = Managed {
    handle :: H
  , managed :: a
  } deriving (Eq,Show)
```

Now, because we want to react to the fact that an object is being managed – or not managed anymore – we have
to introduce special effects.

## Effectful managing

Hence two new types: `Manager m` and `EffectfulManage a s l`.

### `Manager`

A manager is a monad that can generate new handles to manage any kind of value and recycle managed values:

```haskell
class (Monad m) => Manager m where
  manage :: a -> m (Managed a)
  drop   :: Managed a -> m ()
```

`manage a` will turn `a` into a managed version you can use for whatever you want. In theory, you shouldn’t
have access to the constructor of `Managed` nor the `handle` field.

If a type implements both `Monad` and `Manager`, we can manage values and recycle them very easily:

```haskell
import Prelude hiding ( drop )

foo :: (Manager m) => m ()
foo = do
  x <- manage 3
  y <- manage "hey!"
  drop x
  drop y
```

**Notice**: if you want to be able to use the `drop` function, you’ll have to hide `Prelude`’s `drop`.

### `EffectfulManage`

However, we’d like to be able to react to the fact a value is now tracked by our monad, or recycled. That’s
done through the following typeclass:

```haskell
class EffectfulManage a s l | a -> s l where
  spawned :: Managed a -> s
  lost    :: Managed a -> l
```

`EffectfulManage a s l` provides an event `s` and an event `l` for `a`. If you’re not comfortable with functional
dependencies, `a -> s l` means you can’t have two pairs of events for the same type.

Let’s take an example.

```haskell
data IntSpawned = IntSpawned (Managed Int)
data IntLost = IntLost (Managed Int)

instance EffectfulManage Int IntSpawned IntLost where
  spawned = IntSpawned
  lost = IntLost
```

Pretty simple. Now, there’re two functions to react to such events:

```haskell
spawn :: (Manager m,EffectfulManage a s l,Effect s m) => a -> m (Managed a)
lose :: (Manager m,EffectfulManage a s l,Effect l m) => Managed a -> m ()
```

`spawn a` manages the value `a`, returning its *managed* version, and as you can see in the type signature, have
an effect of type `s`, which is the *spawned* effect. `lost a` takes a managed value, drops it, and emits the
corresponding `l` event.

In our case, with our `Int`, we can specialize both the functions this way:

```haskell
spawn :: (Manager m,EffectfulManage Int IntSpawned IntLost,Effect IntSpawned m) => Int -> m (Managed Int)
lost :: (Manager m,EffectfulManage Int IntSpawned IntLost,Effect IntLost m) => Managed Int -> m ()
```

Let’s write a simple example:

```haskell
instance (Functor m,Monad m) => Manager (StateT [H] m) where
  manage x = fmap (flip Managed x) (gets head <* modify tail)
  drop (Managed h _) = modify $ (:) h

-- a possible backend...
instance Effect IntSpawned (StateT [H] IO) where
  react (IntSpawned (Managed h i)) = liftIO . putStrLn $ "int has spawned:" ++ show i

instance Effect IntLost (StateT [H] IO) where
  react (IntLost (Managed h i)) = liftIO . putStrLn $ "int lost :" ++ show i
```

# In the end

This is being implemented in **photon**, and I think it’s a good start. I once wanted to use **pipes**, but
a 3D engine is not a streaming problem: it’s a reactive problem. Maybe FRP could be more elegant, I don’t
know – the `H` type is not the most elegant thing ever, but it works fine.

What do you think folks?



[^FRP]: Functional Reactive Programming
