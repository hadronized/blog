On the weekend of 14th – 17th of April 2017, I was attending for the forth time the
[easter demoparty Revision 2017](https://2017.revision-party.net). This demoparty is the biggest so
far in the world and gathers around a thousand people coming from around the world. If you’re a
demomaker, a demoscene passionated or curious about it, that’s the party to go. It hosts plenty of
competitions, among *photos*, *modern graphics*, *oldschool graphics*, *games*, *size-limited demos*
(what we call *intros*), *demos*, *tracked and streamed music*, etc. It’s massive.

So, as always, once a year, I attend Revision. But this year, it was a bit different for me. Revision
is *very* impressive and most of the *“big demogroups”* release their productions they’ve been
working on for months or even years. I tend to think *“If I release something here, I’ll just be
kind of muted by all those massive productions.”* Well, less than two weeks before Revision 2017, I
was contacted by another demogroup. They asked me to write an *invitro* – a kind of *intro* or
*demo* acting as a communication production to invite people to go to another party. In my case, I
was proposed to make the [Outline 2017](http://outlinedemoparty.nl) invitro. Ouline was the first
party I attended years back then, so I immediately accepted and started to work on something. That
was something like 12 days before the Revision deadline.

I have to tell. It was a challenge. All productions I wrote before was made in about a month and a
half and featured less content than the Outline Invitation. I had to write a lot of code from
scratch. *A lot*. But it was also a very nice project to test my demoscene framework, written in
Rust – you can find [spectra here](https://github.com/phaazon/spectra) for now; it’s unreleased by
the time I’m writing this but I plan to push it onto crates.io very soon.

An hour before hitting the deadline, the beam team told me their Ubuntu compo machine died and that
it would be neat if I could port the demo to Windows. I rushed like a fool to make a port – I even
forked and modified my OpenAL dependency! – and I did it in 35 minutes. I’m still a bit surprised
yet proud!

Anyways, this post is not about bragging. It’s about hindsight. It’s a post-mortem. I did that for
[Céleri Rémoulade](http://www.pouet.net/prod.php?which=67966) as I was the only one working on it –
music, gfx, direction and obviously the Rust code. I want to draw a list of *what went wrong* and
*what went right*. In the first time, for me. So that I have enhancement axis for the next set of
demos I’ll make. And for sharing those thoughts so that people can have a sneak peek into the
internals of what I do mostly – I do a lot of things! :D – as a hobby on my spare time.

# What went wrong

Sooooooooo… What went wrong. Well, a lot of things! **spectra** was designed to build demo
productions in the first place, and it’s pretty good at it. But something that I have to enhance is
the *user interaction*. Here’s a list of what went wrong in a concrete way.

## Hot-reloading went wiiiiiiiiiiild²

With that version of **spectra**, I added the possibility to *hot-reload* almost everything I use as
a resource: shaders, textures, meshes, objects, cameras, animation splines, etc. I edit the file,
and as soon as I save it, it gets hot-reloaded in realtime, without having to interact with the
program (for curious ones, I use the straight-forward [notify crate](https://crates.io/crates/notify)
crates for registering callbacks to handle file system changes). This is very great and it saves a
**lot** of time – Rust compilation is slow, and that’s a lesson I’ve learned from Céleri Rémoulade:
keeping closing the program, making a change, compiling, running is a waste of time.

So what’s the issue with that? Well, the main problem is the fact that in order to implement
hot-reloading, I wanted performance and something very simple. So I decided to use *shared mutable
smart states*. As a **Haskeller**, I kind of offended myself there – laughter! Yeah, in the
Haskell world, we try hard to avoid using shared states – `IORef` – because it’s not referentially
transparent and reasoning about it is difficult. However, I tend to strongly think that in some very
specific cases, you need such side-effects. I’m balanced but I think it’s the way to go.

Anyway, in Rust, shared mutable state is implemented via two types: `Rc/Arc` and `Cell/RefCell`.

The former is a runtime implementation of the Rust *borrowing rules* and enables you to share a
value. The borrowing rules are not enforced at compile-time anymore but dynamically checked. It’s
great because in some cases, you can’t know how long your values will be borrowed or live. It’s also
dangerous because you have to pay extra attention to how you borrow your data – since it’s checked
at runtime, you can crash your program if you’re not extra careful.

> `Rc` means *ref counted* and `Arc` means *atomic-ref counted*. The former is for values that stay
> on the same and single thread; the latter is for sharing between threads.

`Cell/RefCell` are very interesting types that provide *internal mutation*. By default, Rust gives
you *external mutation*. That is, if you have a value and its address, can mutate what you have at
that address. On the other hand, *internal mutation* is introduced by the `Cell` and `RefCell`
types. Those types enable you to mutate the content of an object stored at a given address without
having the exterior mutation property. It’s a bit technical and related to Rust, but it’s often
used to mutate the content of a value via a function taking an immutable value. Imagine an immutable
value that only holds a pointer. Exterior mutation would give you the power to change what this
pointer points to. Interior mutation would give you the power to change the object pointed by this
pointer.

> `Cell` only accepts values that can be copied bit-wise and `RefCell` works with references.

Now, if you combine both – `Rc<RefCell<_>>`, you end up with a shareable – `Rc<_>` – mutable –
`RefCell<_>` – value. If you have a value of type `Rc<RefCell<u32>>` for instance, that means you
can clone that integer and store it everywhere in the same thread, and at any time, borrow it and 
inspect and/or mutate it. All copies of the values will observe the change. It’s a bit like C++’s
`shared_ptr`, but it’s safer – thank you Rust!

So what went wrong with that? Well, the borrow part. Because Rust is about safety, you still need to
tell it how you want to borrow at runtime. This is done with the [`RefCell::borrow()`](https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.borrow]
and [`RefCell::borrow_mut()`](https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.borrow_mut)
functions. Those functions return special objects that borrow the ref as long as it lives. Then,
when it goes out of scope, it releases the borrow.

So any time you want to use an object that is hot-reloadable with my framework, you have to call
one of the borrow functions presented above. You end up with a lot of borrows, and you have to keep
in mind that you can litterally crash your program if you violate the borrowing rules. This is a
nasty issue. For instance, consider:

```rust
let cool_object = …; // Rc<RefCell<Object>>, for instance
let cool_object_ref = cool_object.borrow_mut();
// mutate cool_object

just_read(&cool_object.borrow()); // borrowing rule violated here because a mut borrow is in scope
```

As you can see, it’s pretty simple to fuck up the program if you don’t pay extra attention to what
you’re doing with your borrow. To solve the problem above, you’d need a smaller scope for the
mutable borrow:

```rust
let cool_object = …; // Rc<RefCell<Object>>, for instance

{
  let cool_object_ref = cool_object.borrow_mut();
  // mutate cool_object
}

just_read(&cool_object.borrow()); // borrowing rule violated here because a mut borrow is in scope
```

So far, I haven’t really spent time trying to fix that, but that’s something I have to figure out.

## Resources declaration in code

This is a bit tricky. As a programmer, I’m used to write algorithms and smart architectures to
transform data and resolve problems. I’m given inputs and I provide the outputs – the solutions.
However, a demoscene production is special: you don’t have inputs. You create artsy audiovisual
outputs from nothing but time. So you don’t really write code to solve a problem. You write code
to create something that will be shown on screen or in a headphone. This aspect of demo coding has
an impact on the style and the way you code. Especially in crunchtime. I have to say, I was pretty
bad on that part with that demo. To me, code should only be about transformations – that’s why I
love Haskell so much. But my code is clearly not.

If you know the `let` keyword in Rust, well, imagine hundreds and hundreds of lines starting with
`let` in a single function. That’s most of my demo. In rush time, I had to declare a *lot* of things
so that I can use them and transform them. I’m not really happy with that, because those were data
only. Something like:

```rust
let outline_emblem = cache.get::<Model>("Outline-Logo-final.obj", ()).unwrap();
let hedra_01 = cache.get::<Model>("DSR_OTLINV_Hedra01_Hi.obj", ()).unwrap();
let hedra_02 = cache.get::<Model>("DSR_OTLINV_Hedra02_Hi.obj", ()).unwrap();
let hedra_04 = cache.get::<Model>("DSR_OTLINV_Hedra04_Hi.obj", ()).unwrap();
let hedra_04b = cache.get::<Model>("DSR_OTLINV_Hedra04b_Hi.obj", ()).unwrap();
let twist_01 = cache.get::<Object>("DSR_OTLINV_Twist01_Hi.json", ()).unwrap();
```

It’s not that bad. As you can see, **spectra** features a *resource cache* that provides several
candies – hot-reloading, resource dependency resolving and resource caching. However, having to
declare those resources directly in the code is a nasty boilerplate to me. If you want to add a new
object in the demo, you have to turn it off, add the Rust line, re-compile the whole thing, then run
it once again. It breaks the advantage of having hot-reloading and it pollutes the rest of the code,
making it harder to spot the actual transformations going on.

This is even worse with the way I handle texts. It’s all `&'static str` declared in a specific file
called `script.rs` with the same insane load of `let`. Then I rasterize them in a function and use
them in a very specific way regarding the time they appear. Not fun.

## Still not enough data-driven

As said above, the cache is a great help and enables some data-driven development, but that’s not
enough. The `main.rs` file is more than 600 lines long and 500 lines are just declarations of of 
clips (editing) and are all very alike. I intentionally didn’t use the runtime version of the
timeline – but it’s already implemented – because I was editing a lot of code at that moment, but
that’s not a good excuse. And the timeline is just a small part of it (the cuts are something like
10 lines long) and it annoyed me at the very last part of the development, when I was synchronizing
the demo with the soundtrack.

I think the real problem is that the clips are way too abstract to be a really helpful abstraction.
Clips are just lambdas that consume time and output a node. This also has implication (you cannot
borrow something for the node in your clip because of borrowing rules ; duh!).

## Animation edition

Most of the varying things you can see in my demos are driven by animation curves – splines. The
bare concept is very interesting: an animation contains control points that you know have a specific
value at a given time. Values in between are interpolated using an interpolation mode that can
change at each control points if needed. So, I use splines to animate pretty much everything: camera
movements, objects rotations, color masking, flickering, fade in / fade out effects, etc.

Because I wanted to be able to edit the animation in a comfy way – lesson learned from Céleri
Rémoulade, splines can be edited in realtime because they’re hot-reloadable. They live in JSON files
so that you just have to edit the JSON objects in each file and as soon as you save it, the
animation changes. I have to say, this was very ace to do. I’m so happy having coded such a feature.

However, it’s JSON. It’s already a thing. Though, I hit a serious problem when editing the
orientations data. In **spectra**, an orientation is encoded with a
[unit quaternion](https://en.wikipedia.org/wiki/Quaternion#Unit_quaternion). This is a 4-floating
number – hypercomplex. Editing those numbers in a plain JSON file is… challenging! I think I really
need some kind of animation editor to edit the spline.

## Video capture

The Youtube [capture](https://www.youtube.com/watch?v=OemyLQbDTSk) was made directly in the demo.
At the end of each frame, I dump the frame into a .png image (with a name including the number of
the frame). Then I simply use ffmpeg to build the video.

Even though this is not very important, I had to add some code into the production code of the demo
and I think I could just refactor that into **spectra**. I’m talking about three or four lines of
code. Not a big mess.

## Compositing

This is will appear as both pros. and cons. Compositing, in spectra, is implemented via the
concept of *nodes*. A node is just an algebraic data structure that contains *something* that can
be connected to *another thing* to compose a render. For instance, you get find nodes of type
*render*, *color*, *texture*, *fullscreen effects* and *composite* – the latter is used to mix
nodes between them.

Using the nodes, you can build a tree. And the cool thing is that I implemented the most common
operators from `std::ops`. I can then apply a simple color mask to a render by doing something like

```rust
render_node * RGBA::new(r, g, b, a).into()
```

This is extremely user-friendly and helped me a lot to tweak the render (the actual ASTs are more
complex than that and react to time, but the idea is similar). However, there’s a problem. In the
actual implementation, the *composite* node is not smart enough: it blends two nodes by rendering
them into a separate framebuffer (hence two framebuffers), then sample via a fullscreen quad the
left framebuffer and then the right one – and apply the appropriate blending.

I’m not sure about performance here, but I feel like this is the wrong way to go – bandwidth! I
need to profile.

## On a general note: data vs. transformations

My philosphy is that code should be about transformation, not data. That’s why I love Haskell.
However, in the demoscene world, it’s very common to move data directly into functions – think of
all those fancy shaders you see everywhere, especially on shadertoy. As soon as I see a data
constant in my code, I think “Wait; isn’t there a way to remove that from the function and have
access to it as an input?”

This is important, and that’s the direction I’ll take from now on for the future versions of my
frameworks.

# What went right!

A lot as well!

## Hot reloading

Hot reloading was *the* thing I needed. A hot-reload everything. I even hot-reload the tessellation
of the objects (.obj), so that I can change the shape / resolution of a native object and I don’t
have to relaunch the application. I saved a lot of precious time thanks to that feature.

## Live edition in JSON

I had that idea pretty quickly as well. A lot of objects – among splines – live in JSON files. You
edit the file, save it and tada: the object has changed in the application – hot reloading! The JSON
was especially neat to handle splines of positions, colors and masks – it went pretty bad and wrong
with orientations, but I already told you that.

## Compositing

As said before, compositing was also a win, because I lifted the concept up to the Rust AST,
enabling me to express interesting rendering pipeline just by using operators like `*`, `+` and
some combinators of my own (like `over`).

## Editing

Editing was done with a cascade of types and objects:

- a `Timeline` holds several `Track`s and `Overlap`s;
- a `Track` holds several `Cuts`;
- a `Cut` holds information about a `Clip`: when the cut starts and ends in the clip and when such a
  cut should be placed in the track;
- a `Clip` contains code defining a part of the scene (Rust code, can’t live in JSON for obvious
  reasons;
- an `Overlap` is a special object used to fold several nodes if several cuts are triggered at the
  same time; it’s used for transitions mostly;
- alternatively, a `TimelineManifest` can be used to live-edit all of this (the JSON for the cuts
  has a string reference for the clip, and a map to actual code must be provided when folding the
  manifest into an object of type `Timeline`).

I think such a system is very neat and helped me a lot to remove naive conditions (like timed
if-else if-else if-else if…-else nightmare). With that system, there’s only one test per frame to
determine which cuts must be rendered (well, actually, one per track), and it’s all declarative.
Kudos.

## Resources loading was insanely fast

I thought I’d need some kind of loading bars, but everything loaded so quickly that I decided it’d
be a wast of time. Even though I might end up modifying the way resources are loaded, I’m pretty
happy with it.

# Conclusion

Writing that demo in such a short period of time – I have a job, a social life, other things I
enjoy, etc. – was such a challenge! But it was also the perfect project to stress-test my framework.
I saw a lot of issues while building my demo with spectra and a lot of “woah, that’s actually pretty
great doing this that way!”. I’ll have to enhance a few things, but I’ll always do that with a demo
as a *work-in-progress* because I target pragmatism. As I released spectra on crates.io, I might end
up writing another blog entry about it sooner or later, and even speak about it at a Rust meeting!

Keep the vibe!
