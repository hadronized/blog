As a programmer, you might already have come across **side- effects**. You haven’t? Let’s have a look at these nasty
things.

## Side effect

A **side effect** is a big word to describe an *effect* a function has that mutates something out of scope. For
instance, you could picture a function modifying a global state, environment or any kind of “*remote*” value.

Another way to understand what a side effect is is to look at referencial
[transparency](http://en.wikipedia.org/wiki/Referential_transparency_%28computer_science%29). If you
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
objects that code might touch. It could be class method, it could be a simple function, `_x` and `_y` could be `int`
as well as they could be `Foo` objects with the `operator+=` overloaded. The list is long.

In that snippet, `_x += rx` and `_y += ry` are side effects, because they alter “something” *elsewhere*. That could
be fine, but it’s not. The more you have side effects, the harder it is to debug, maintain, compose, understand and
make your code base evolve. As a good programmer, you should care about side effect avoidance.

## Purity

In pure functional language like **Haskell*, we can’t do that, at least not directly. We would write this function:

```
update :: (Int,Int) -> (Int,Int) -> (Int,Int)
update (x,y) (rx,ry) = (x + rx,y + ry)
```

Because everything is immutable in **Haskell**, we are sure that `(x,y)` and `(rx,ry)` are constant. They **can’t**
change. So we can say what that function does:

> *It takes a pair of int and apply them an offset!*

Yeah, exactly. And it can’t be **anything else**. We don’t need assumptions, because such a function is
*transparent*. That’s a great property you can rely on – do it, your compiler does it ;)

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
  
  void addObserver(Observer &observer) {
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
Being able to compose my functions and have effects.

In [photon](https://github.com/phaazon/photon), my 3D engine, I use explicit effects.
