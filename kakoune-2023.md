Oh my… 2023 has been such a trek so far… If you have missed my [previous article](https://github.com/helix-editor/helix)
about my thoughts about editors and development platforms, I think it’s probably the moment to have a look at it.

Today is the end of May 2023. [Helix](https://helix-editor.com/) has been my primary editor for months now. I haven’t
come back to [Neovim](https://neovim.io/). In the previous article, I mentioned that I couldn’t give a fair opinion
about Helix because I had just started using it. Today, I think I have enough experience and usage (5 / 6 months) to
share what I think about Helix, and go a little bit further, especially regarding software development in general and,
of course, editing software.

However, before starting, I think I need to make a clear disclaimer. If you use Vim, Neovim, Emacs, VS Code, Sublime
Text, whatever, and you think that _“Everyone should just use whatever they want”_, then we agree and **this is not
the point of this blog article**. The goal is to discuss a little bit more than just speaking out obvious takes, but
please do not start waving off the topic because you think _“Everyone should use what they want”_. There is a place
for constructive debate even there. If you start arguing, then it means you want to debate, and then you need to be
ready to have someone with different arguments that will not necessarily go your way, nor your favorite editor.

Finally, if you think I haven’t used Vim / Neovim enough, just keep in mind that I have been using (notice the tense)
Vim (and by extension, Neovim) since I was 15, and I’m 31, so 16 years.

I have wrote that blog article already three times before deleting everything and starting over. I think I will make
another blog article about how I think about software, but here I want to stay focused on one topic: editors and
development environments.

# Helix

From a vimmer perspective, Helix is weird. It reverses the verb and the motion (you don’t type `di(` to delete inside
parenthesis, but you type `mi(d` to first match the parenthesis and then delete). Then, you have the multi-selections
as a default way of doing things; Helix doesn’t really have the concept of a _cursor_, which is a design it gets from
[Kakoune](https://kakoune.org/), and I’ll explain what it means and implies later.

Some vimmers publicly talked about it. ThePrimeagen, for instance, made a Youtube video about it where he basically
just scratched the surface of the editor and _“Hell it’s not Vim so it’s not really good”_. He moved the video to
private then but I’m sure you can just look for Tweets. Many Vim afficionados react that way, which is not a very
serious way of trying out something new, especially if you happen to publicly talk about it to thousands of people. I
think it’s not fair to both the tool (here, Helix) and the people reading you, unless you are one of those Vim zealots
thinking you know everything better than anyone else and dismissing people’ point of views just because they are not
the same as yours.

Anyway, that reputation didn’t hold me from trying out, and the way I try software is simple: just give in and accept
to drop your habits and productivity. Of course, at work, I was still using Vim / Neovim, but switched to Helix on my
spare-time projects.

There are many aspects to talk about with Helix. The first one is that, contrary to what people who haven’t really and
seriously tried it, the difference with Vim is not only “reversed motion/verb and multi-cursors.” The first differenc
is the _design_ and the _direction_.

## Design and direction

Helix is an editor that natively integrates many features that are most of the time plugins in others editors (even
in VS Code). The best example here is [tree-sitter](https://tree-sitter.github.io/tree-sitter/), surrounding pairs (
being able to automatically surround regions of text with `(`, `[`, HTML tags, etc.), LSP UIs, fuzzy pickers, etc. This
is _massive_ and important difference because of two main things:

1. Code maintenance.
2. User experience.

About code maintenance, having all of those features natively integrated in the editor means that you are 100% sure that
if you get the editor to start, the features will work, and the editor developers will keep that updated within the
next iteraton of the editor. For instance, having tree-sitter natively implemented (Rust) in Helix means that the
editor itself knows about tree-sitter and its grammars, highlights queries and features. The same thing goes for
surrounding-pairs or auto-pairs, for instance. If the team decides to change the way text is handled in buffers, then
the code for auto-pairs / surrounding-pairs **will have to be updated for the editor to be releasable**.

The user experience will then be better, because you get those nice features without having to start looking around
for a plugin doing it, with the problem of chosing the right one among a set of competing plugins. Plus the risk of
having your plugin break because it’s not written by the Helix team. Just install the software and start using those
features.

For now, Helix ships with a bunch of powerful features that makes it usable in a modern environment:

- tree-sitter support for semantic highligthing, semantic selection and navigation, etc. That also includes a native
  support for getting grammars / queries from the CLI with `hx --grammar`.
- LSP support, both in terms of features (go-to, references, implementations, incoming / outgoing calls, diagnostics,
  inlay hints, etc. etc.) and in terms of UI. It’s the same UI for everyone.
- Surrounding pairs.
- Auto pairs.
- Git integration (gutter diff on the right side of your buffer) and Git semantic object (go to next change, etc.).
- Registers and user-defined registers (like in Vim), along with macros (but you won’t need them, trust me).
- Native discoverability, including a command palette with _every possible available commands_, tagged with their
  keybindings.
- Many bundled themes (yes, `catppuccin` themes are there!).
- Snappier than anything else.
- Various integration with your system, including clipboards, external commands, etc.

There is also one (major) thing I want to talk about and that deserves its own section: configuration.

## Configuration done right

Helix treats configuration _the way it should be_: as data. Configuration in Helix is done in TOML. There is no
scripting implied, it’s only a rich object laid out as TOML sections. And this is a _joy_ to use. For instance, this is
my current Helix configuration (excluding keys remapping, because my keyboard layout is bépo):

```toml
theme = "catppuccin_macchiato"

[editor]
scroll-lines = 1
cursorline = true
auto-save = false
completion-trigger-len = 1
true-color = true
color-modes = true
auto-pairs = true
rulers = [120]
idle-timeout = 50

[editor.cursor-shape]
insert = "bar"
normal = "block"
select = "underline"

[editor.indent-guides]
render = true
character = "▏"

[editor.lsp]
display-messages = true
display-inlay-hints = true

[editor.statusline]
left = ["mode", "spinner", "file-name", "file-type", "total-line-numbers", "file-encoding"]
center = []
right = ["selections", "primary-selection-length", "position", "position-percentage", "spacer", "diagnostics", "workspace-diagnostics", "version-control"]
```

Notice the tree-sitter and LSP configuration. Yes. _None_.

This is so important to me. Because configuration is data, it is simple for Helix to expose it and present it to the 
user by reading the TOML file without caring about having any side-effects. Helix has a command line (`:`) where you can
tweak those options dynamically. And you can dynamically reload the configuration as well.

## Multi-selection centric

The major difference, even before the reversed motion/verb thing, is the fact that Helix _doesn’t really_ have a cursor.
It has the concept of _selections_. A selection is made of two entities:

- An _anchor_.
- A _cursor_.

The cursor is the part of the selection that moves when you extend the selection. The anchor, as the name implies, is
the other part that stays where it is: it’s anchored. **By default, you have only one selection** and the anchor is 
located at the same place as the cursor. It looks similar to any other editor. Things start to change when you begin
typing normal commands. Typing `l`, for instance, will move both the anchor and cursor to the right, making them a
single  visual entity. However, if you type `w`, the cursor will move to the end of the word while the anchor will move
to its beginning, visually selecting the word. If you type `W`, the anchor won’t move and only the cursor will move,
extending the selection. If you press `B`, it will move the cursor back one word, shrinking the selection. You can press
`<a-;>` to flip the anchor and the selection, which is useful when you want to extend on the left or on the right.

This concept of selection is really powerful because everything else is based on it. Pressing `J` will move the cursor
down one line, leaving the anchor on the current line, extending selected lines. Once something is selected, you can
operate on it, with `d`, `c`, `y`, `r`, etc. For instance, `wd` will select the word and delete it. `Jc` will extend the
selection with the next line and start changing. Selections in Helix are not just visual helps: they represent what
normal editing operations will work on, which is a great design, because you can extend, shrink and reason about them in
a much more seamless and natural way.

But it’s just starting. Remember earlier when I say that **by default, you have only one selection**? Well, you can have
many, and this is where Helix starts to really shine to me. The first way to create many selections is to press `C`. `C`
will duplicate your current selection on the next line. If you have the anchor at the same position as the cursor,
pressing `C` will make it like you have another cursor on the next line below. For instance, consider this text:

```text
I love su|shi.
But I also love pizza.
```

The `|` is our cursor (but also anchor). If you press `C`, you will see something like this:

```text
I love su|shi.
But I als|o love pizza.
```

But as I had mentioned, `C` duplicates _selections_. Let’s say you started like this, `<` being the anchor and `|` the
cursor:

```text
I love <sus|hi.
But I also love pizza.
```

Pressing `C` now will do this:


```text
I love <sus|hi.
But I a<lso| love pizza.
```

Once you have many selections, everything you type as normal commands will be applied to every selections. If I type
`f.`, it will set the anchor to the current cursor and extend the cursor to the next `.`, resulting in this:

```text
I love sus<hi|.
But I also< love pizza|.
```

Press `<a-;>` to swap anchors and cursors:

```text
I love sus|hi<.
But I also| love pizza<.
```

And then pressing `B` will do this:

```text
I love |sushi<.
But I |also love pizza<.
```

Multi-cursor is then not only a nice visual help, but also a completely new way of editing your buffers. Once you get
the hang of it, you don’t really think in terms of a single cursor but many selections, ranges, however you like to call
them.

## Getting more cursors

`C` is great, but this not something we usually use. Instead, we use features that don’t exist in Vim. I’m not entirely
sure how to call those, but I like to call them _selection generators_. They come in many different flavors, so I’ll
start with the easiest one and will finish with the most interesting and (maybe a bit?) obscure at first.

### Matching matching matching!

The `m` key is Helix is a wonderful key. It’s the _match_ key. It expects a motion and will change all your selections
to match the motion. For instance, `mia` _“matches inside arguments”_ (tree-sitter). Imagine this context:

```rust
fn foo(x: i32, y: |i32) {}

fn bar(a: String, |b: bool) {}
```

Press `mia` to get this:

```rust
fn foo(x: i32, <y: i32|) {}

fn bar(a: String, <b: bool|) {}
```

Then, for instance, press `<a-;>` to flip the anchor and the cursor:

```rust
fn foo(x: i32, |y: i32<) {}

fn bar(a: String, |b: bool<) {}
```

`F,` to select the previous `,`:

```rust
fn foo(x: i32|, y: i32<) {}

fn bar(a: String|, b: bool<) {}
```

And just press `d`:

```rust
fn foo(x: i32) {}

fn bar(a: String) {}
```

It’s so logical, easy to think about and natural. Someting interesting to notice, too, is that contrary to Vim, which
has many keys doing mainly the same thing, making things weird and not really well designed. For instance, `vd` selects
the current character and delete it. `vc` deletes the current character and puts you into insert mode. `s` does exactly 
the same. Why would you have a key in direct access doing something so specific? If you want to delete a word, you 
either press `vwd`, or more simply in Vim, `dw`. All of that is already confusing, but it doesn’t end there. `x` deletes
the current character, but `x` is actually _cut_, so if you select a line with `V` and press `x`, it will cut the line.
Press `Vc` to change a line… or just `S`. What?

All those shortcuts feel like exceptions you have to learn, and it’s a good example of a flawed design. On the other
side, Helix (which is actually a Kakoune design it’s based on), have a single character to delete something: `d`. Since
the editor has multi-selections as a native editing entity, all of those situations will imply using the `d` key:

- Deleting the current character: `d`.
- Deleting the next word: `wd`.
- Deleting the current line: `xd` (`x` selects and extend the line).
- Deleting delimiters but not the content: `md(`.
- Etc. etc.

That applies to everything.

### Selecting, splitting, keeping and removing

The design of editing in Helix (Kakoune) is to be interactive and iterative. For instance, consider the following:

```rust
  pub fn new(
    line_start: usize,
    col_start: usize,
    line_end: usize,
    col_end: usize,
    face: impl Into<String>,
  ) -> Self {
    Self {
      line_start,
      col_start,
      line_end,|
      col_end,
      face: face.into(),
    }
  }
```

Let’s say we would like, to begin with, select very quickly every arguments type and switch them to `i32`. Many ways
of doing that, but let’s see one introducing a great concept: _selecting_. Selecting allows you to create new 
selections that satisfy a regex. The default keybinding for that is `s` for… select (woah). `s` always applies to the 
current Selections (notice the use of plural, it will be useful later). We then need to start selecting something. Here,
we can just press `mip` to select inside the paragraph, since our cursor is right after `line_end`:

```rust
< pub fn new(
    line_start: usize,
    col_start: usize,
    line_end: usize,
    col_end: usize,
    face: impl Into<String>,
  ) -> Self {
    Self {
      line_start,
      col_start,
      line_end,
      col_end,
      face: face.into(),
    }
  }|
```

We have the whole thing selected. Press `s` to start selecting with a regex. We want the arguments, so let’s select
everything with `:` and press `s:` and return:

```rust
  pub fn new(
    line_start<:| usize,
    col_start<:| usize,
    line_end<:| usize,
    col_end<:| usize,
    face<:| impl Into<String>,
  ) -> Self {
    Self {
      line_start,
      col_start,
      line_end,
      col_end,
      face<:| face.into(),
    }
  } 
```

See how it created a bunch of selections for us. Also, notice that it selected `face: face.into()`, which is not
correct. We want to _remove_ that selection. Again, several ways of doing it. Something to know is that, Helix (Kakoune)
has the concept of _primary_ selection. This is basically the selection on which you are going to apply actions first,
like LSP hover, etc (it would be a madness to have LSP hover applies to all selections otherwise!). You can cycle the
primary selection with `(` and `)`. Once you reach the one you want, you can press `<a-,>` to just drop the selection.
However, we don’t want to cycle things. We want a faster way.

Let’s talk about _removing_ selections. The default keybinding is `<A-K>`. However, here, our selections are all
about the same content (the `:`). As mentioned before, pressing `x` will select the current line of every selections:

```rust
  pub fn new(
<   line_start: usize,|
<   col_start: usize,|
<   line_end: usize,|
<   col_end: usize,|
<   face: impl Into<String>,|
  ) -> Self {
    Self {
      line_start,
      col_start,
      line_end,
      col_end,
<     face: face.into(),|
    }
  } 
```

Let’s filter selection and remove the one matching a pattern. In our case, let’s remove selections with a
`(` in them: `<A-K>\(` and return:

```rust
  pub fn new(
<   line_start: usize,|
<   col_start: usize,|
<   line_end: usize,|
<   col_end: usize,|
<   face: impl Into<String>,|
  ) -> Self {
    Self {
      line_start,
      col_start,
      line_end,
      col_end,
      face: face.into(),
    }
  } 
```

Now press `_` to shring the selections to trim leading and trailing whitespaces:

```rust
  pub fn new(
    <line_start: usize,|
    <col_start: usize,|
    <line_end: usize,|
    <col_end: usize,|
    <face: impl Into<String>,|
  ) -> Self {
    Self {
      line_start,
      col_start,
      line_end,
      col_end,
      face: face.into(),
    }
  } 
```

Imagine that we have changed our mind and now we actually want to change the `usize` to `i32`. We can use the _keep_
operator, which is bound to `K` by default. Press `Kusize` and return to get this:

```rust
  pub fn new(
    <line_start: usize,|
    <col_start: usize,|
    <line_end: usize,|
    <col_end: usize,|
    face: impl Into<String>,
  ) -> Self {
    Self {
      line_start,
      col_start,
      line_end,
      col_end,
      face: face.into(),
    }
  } 
```

Another possible way is to press `susize` to only select the `usize` directly, which might be wanted if you want to 
change them quickly to `i32`, for instance.
